<!-- ---
layout: post
title:  "记一次内核之旅--修复板载声卡前置放大器驱动"
date:   2023-05-26 13:02:30 +0800
tags: 
	- linux
	- 设备驱动
categories: 
	- linux
toc: true
--- -->
# 记一次内核之旅--修复板载声卡前置放大器驱动

2023-05-26

## 问题
笔记本电脑在Linux下一直没有办法正常使用内置音响，初步判断是声卡驱动的问题。

## 寻找解题
1. 首先通过`Arch Linux Wiki`，偶然间看到一个sof-firmware，安装之后可以正常检测到`Realtek`声卡。耳机孔，蓝牙运作正常。但内置音响声音非常小，且底部低音音响不出声。据此判断，应该是有模拟放大器未激活，导致模拟电路输出功率不足。

<!-- more -->

2. 用`alsa-info`脚本进行信息搜集：用于解码的声卡应该为`Realtek ALC287`， 放大器芯片应该为`Cirrus cs35l41`

3. 进行广泛搜索后，发现了一篇[Gist](https://gist.github.com/lamperez/862763881c0e1c812392b5574727f6ff)，作者的`amp`与我相同。但其解决方案为直接对`BIOS`进行反编译，然后修改`_DSD entry`。作者电脑为ASUS的`Zenbook UX3402`与我不同。且我在对自己的BIOS进行反编译时，发现Lenovo的编码方式无法通过Intel的反编译器完整解析，便作罢。

4. 向[Arch Linux Bug](https://bugs.archlinux.org/task/78565)反馈，求助。按照[Debugging Regressions](https://wiki.archlinux.org/title/Kernel#Debugging_regressions)进行相应测试，无果。于是转战[kernel bugzilla](https://bugzilla.kernel.org/)

5. 结合对[kernel邮件组](https://lore.kernel.org/lkml/c72a2f28a6e5c2c3c9ed17269bd56e7484df960c.camel@ljones.dev/)的潜伏观察，锁定了一个[thread](https://bugzilla.kernel.org/show_bug.cgi?id=216194)。这里有位大佬Cameron Berkenpas自己写了patch并在自己的电脑上进行测试，通过了半年的稳定性测试。而另一位大佬[确认了](https://bugzilla.kernel.org/show_bug.cgi?id=216194#c106)和我相同型号设备也可使用同patch。于是准备自己动手打内核补丁。

## 思路
1. 参照经社区验证的[补丁](https://bugzilla.kernel.org/attachment.cgi?id=303828&action=diff)，进行自己相应的[修改](https://bugzilla.kernel.org/show_bug.cgi?id=216194#c108)。

2. 下载stable源码，打补丁，编译内核及对应内核模块，安装内核和内核模块，生成initramfs，用dkms安装树外模块（nvidia驱动等），最后更新grub配置，重启，进入新内核。

## 步骤
```c
// 修改社区的补丁lenovo-7i-gen7-sound-6.2.0-rc3-0.0.5b-002.patch (https://bugzilla.kernel.org/attachment.cgi?id=303828)
// (https://gist.github.com/levihuayuzhang/6137ae4ae46301a355fd37c63e0d876a)
diff --git a/sound/pci/hda/cs35l41_hda.c b/sound/pci/hda/cs35l41_hda.c
index f7815ee24f83..93d86c5a9d53 100644
--- a/sound/pci/hda/cs35l41_hda.c
+++ b/sound/pci/hda/cs35l41_hda.c
@@ -1270,6 +1270,8 @@ static int cs35l41_hda_read_acpi(struct cs35l41_hda *cs35l41, const char *hid, i
 	size_t nval;
 	int i, ret;
 
+	printk("CSC3551: probing %s\n", hid);
+
 	adev = acpi_dev_get_first_match_dev(hid, NULL, -1);
 	if (!adev) {
 		dev_err(cs35l41->dev, "Failed to find an ACPI device for %s\n", hid);
@@ -1287,8 +1289,9 @@ static int cs35l41_hda_read_acpi(struct cs35l41_hda *cs35l41, const char *hid, i
 	property = "cirrus,dev-index";
 	ret = device_property_count_u32(physdev, property);
 	if (ret <= 0) {
-		ret = cs35l41_no_acpi_dsd(cs35l41, physdev, id, hid);
-		goto err_put_physdev;
+	    //ret = cs35l41_no_acpi_dsd(cs35l41, physdev, id, hid);
+	    //goto err_put_physdev;
+	    goto no_acpi_dsd;
 	}
 	if (ret > ARRAY_SIZE(values)) {
 		ret = -EINVAL;
@@ -1383,6 +1386,92 @@ static int cs35l41_hda_read_acpi(struct cs35l41_hda *cs35l41, const char *hid, i
 	put_device(physdev);
 
 	return ret;
+
+no_acpi_dsd:
+	/*
+	 * Device CLSA0100 doesn't have _DSD so a gpiod_get by the label reset won't work.
+	 * And devices created by i2c-multi-instantiate don't have their device struct pointing to
+	 * the correct fwnode, so acpi_dev must be used here.
+	 * And devm functions expect that the device requesting the resource has the correct
+	 * fwnode.
+	 */
+
+	printk("CSC3551: no_acpi_dsd: %s\n", hid);
+
+	/* TODO: This is a hack. */
+	if (strncmp(hid, "CSC3551", 7) == 0) {
+	    goto csc3551;
+	}
+
+	if (strncmp(hid, "CLSA0100", 8) != 0)
+		return -EINVAL;
+
+	/* check I2C address to assign the index */
+	cs35l41->index = id == 0x40 ? 0 : 1;
+	cs35l41->hw_cfg.spk_pos = cs35l41->index;
+	cs35l41->channel_index = 0;
+	cs35l41->reset_gpio = gpiod_get_index(physdev, NULL, 0, GPIOD_OUT_HIGH);
+	cs35l41->hw_cfg.bst_type = CS35L41_EXT_BOOST_NO_VSPK_SWITCH;
+	hw_cfg->gpio2.func = CS35L41_GPIO2_INT_OPEN_DRAIN;
+	hw_cfg->gpio2.valid = true;
+	cs35l41->hw_cfg.valid = true;
+	put_device(physdev);
+
+	return 0;
+
+ csc3551:
+
+	printk("CSC3551: id == 0x%x\n", id);
+
+	// cirrus,dev-index
+	if(id == 0x40)
+	    cs35l41->index = 0;
+	else
+	    cs35l41->index = 1;
+
+	cs35l41->channel_index = 0;
+
+	cs35l41->reset_gpio = gpiod_get_index(physdev, NULL, cs35l41->index, GPIOD_OUT_LOW);
+
+	printk("CS3551: reset_gpio == 0x%x\n", cs35l41->reset_gpio);
+
+	// cirrus,speaker-position
+	if(cs35l41->index == 0)
+	    hw_cfg->spk_pos = 0;
+	else
+	    hw_cfg->spk_pos = 1;
+
+	// cirrus,gpio1-func
+	hw_cfg->gpio1.func = 1;
+        hw_cfg->gpio1.valid = true;
+
+	// cirrus,gpio2-func
+	hw_cfg->gpio2.func = 0x02;
+        hw_cfg->gpio2.valid = true;
+
+	// cirrus,boost-peak-milliamp
+	hw_cfg->bst_ipk = -1;
+
+	// cirrus,boost-ind-nanohenry
+	hw_cfg->bst_ind = -1;
+
+	// cirrus,boost-cap-microfarad
+	hw_cfg->bst_cap = -1;
+
+	cs35l41->speaker_id = cs35l41_get_speaker_id(physdev, cs35l41->index, nval, -1);
+
+        if (hw_cfg->bst_ind > 0 || hw_cfg->bst_cap > 0 || hw_cfg->bst_ipk > 0)
+                hw_cfg->bst_type = CS35L41_INT_BOOST;
+        else
+                hw_cfg->bst_type = CS35L41_EXT_BOOST;
+
+	hw_cfg->valid = true;
+
+	put_device(physdev);
+
+	printk("CSC3551: Done.\n");
+
+	return 0;
 }
 
 int cs35l41_hda_probe(struct device *dev, const char *device_name, int id, int irq,
diff --git a/sound/pci/hda/patch_realtek.c b/sound/pci/hda/patch_realtek.c
index e103bb3693c0..8ec2b0f99d8c 100644
--- a/sound/pci/hda/patch_realtek.c
+++ b/sound/pci/hda/patch_realtek.c
@@ -9734,6 +9734,7 @@ static const struct snd_pci_quirk alc269_fixup_tbl[] = {
 	SND_PCI_QUIRK(0x17aa, 0x3853, "Lenovo Yoga 7 15ITL5", ALC287_FIXUP_YOGA7_14ITL_SPEAKERS),
 	SND_PCI_QUIRK(0x17aa, 0x3855, "Legion 7 16ITHG6", ALC287_FIXUP_LEGION_16ITHG6),
 	SND_PCI_QUIRK(0x17aa, 0x3869, "Lenovo Yoga7 14IAL7", ALC287_FIXUP_YOGA9_14IAP7_BASS_SPK_PIN),
+	SND_PCI_QUIRK(0x17aa, 0x38a9, "Lenovo ThinkBook 16p Gen4 IRH", ALC287_FIXUP_CS35L41_I2C_2),
 	SND_PCI_QUIRK(0x17aa, 0x3902, "Lenovo E50-80", ALC269_FIXUP_DMIC_THINKPAD_ACPI),
 	SND_PCI_QUIRK(0x17aa, 0x3977, "IdeaPad S210", ALC283_FIXUP_INT_MIC),
 	SND_PCI_QUIRK(0x17aa, 0x3978, "Lenovo B50-70", ALC269_FIXUP_DMIC_THINKPAD_ACPI),
```

```sh
# https://wiki.archlinux.org/title/Kernel/Traditional_compilation

# download, verifym and extract source code
# modify the patch, then
make mrproper # clean up

zcat /proc/config.gz  > .config # use running kernel config

patch -p1 < ./lenovo-7i-gen7-sound-6.2.0-rc3-0.0.5b-002.patch # applying patch

make -j$(nproc) # max parallel compile

make modules # compile in tree modules

sudo make INSTALL_MOD_STRIP=1 modules_install # install corresponding module

make bzImage # compile kernle image

# copy image to /boot directory and rename it
cp -v arch/x86/boot/bzImage /boot/vmlinuz-linux6.3.4
```

```sh
# make a new mkinitcpio preset for new kernel
cp /etc/mkinitcpio.d/linux.preset /etc/mkinitcpio.d/linux6.3.4.preset

# edit preset of mkinitcpio
vim /etc/mkinitcpio.d/linux6.3.4.preset
-------------------------------------
...
ALL_kver="/boot/vmlinuz-linux6.3.4"
...
default_image="/boot/initramfs-linux6.3.4.img"
...
fallback_image="/boot/initramfs-linux6.3.4-fallback.img"
```

```sh
# generate new initramfs for new kernel
mkinitcpio -p linux6.3.4

# update grub config
sudo grub-mkconfig  -o /boot/grub/grub.cfg 

# out of tree moudle install using dkms
sudo dkms autoinstall -k 6.3.4

# reboot the system and select new kernel at grub entry during booting
reboot now
```

## 总结
1. 这是一个非常危险的patch（仅仅时一个来自社区的workaround而不是fix），未经过Cirrus（硬件制造商）和Lenovo（设备制造商）的官方确认。由社区人员经猜测在另一Levono型号上测试成功，并经猜想应该可以使用到有着相同芯片和类似设计的电脑上。

2. 声卡电路唯一的保护是BIOS中的ACPI控制程序，但恰恰Lenovo在出厂时未在其中合理配置。（Windows中应该是硬编码在了驱动程序中，而Linux社区则还未得到支持）

## 后记
1. 在Cirrus或Lenovo把`quirk`投入内核树前，如果还想继续使用内置音响，必须自己打patch编译内核。（中低音量低音频音响发声，高音量高音频音响发声，应该是存在分频问题）（这导致音响分频不正确，且无法正确调节音量，至少是Gnome中）

2. 或者买个usb音响先凑合

3. 这个问题只能由芯片制造商或者设备制造商解决。因为只有他们才知道芯片的具体设计和相应的接口参数。已在[mail list](https://lore.kernel.org/lkml/b4c202b2-ab29-e2aa-b141-0c967b2c1645@opensource.cirrus.com/)看到Cirrus的工程师了，希望社区能早日得到支持。

## 更新
2024-04-11:

由笔者报告给`Linux mailing list`，并与Cirrus的工程师商讨的第一版fix已由`Stefan Binding <sbinding@opensource.cirrus.com>`提交给`Linux Sound team`,并预计由`V6.8`发布。
1. merge commit: https://github.com/tiwai/sound/commit/37d9d5ff5216df1908a41e6ddd72460c5d938b8a
2. 邮件历史：https://lore.kernel.org/linux-sound/?q=Huayu+Zhang

2024-04-13:

感谢[Stefan大佬的帮助](https://bugzilla.kernel.org/show_bug.cgi?id=218437#c9)，[修复](https://gist.github.com/levihuayuzhang/6137ae4ae46301a355fd37c63e0d876a#file-v6-9-rc1-fix-volumn-control-of-thinkbook-16p-gen4-patch)了音量控制问题。原因是之前的patch使用了解码器和amp之间的错误配置。

2024-04-18:

提交的修复patch进入`Linux Sound`子系统<https://github.com/tiwai/sound/commit/dca5f4dfa925b51becee65031869e917e6229620>

2024-04-22:

patch进入[Linux main line tree](https://github.com/torvalds/linux/commit/dca5f4dfa925b51becee65031869e917e6229620) [v6.9-rc5](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/diff/sound/pci/hda/patch_realtek.c?id=v6.9-rc5&id2=v6.9-rc4).

2024-04-28:

正式发布([v6.8.8](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/diff/sound/pci/hda/patch_realtek.c?id=v6.8.8&id2=v6.8.7))<https://git.kernel.org/stable/ds/v6.8.8/v6.8.7>。

## Ref:
1. Linux kernel关于Cirrus的文档：https://www.kernel.org/doc/Documentation/devicetree/bindings/sound/cirrus%2Ccs35l41.yaml
2. alsa-info输出：http://alsa-project.org/db/?f=1ad9f2709886f0ec5d2d87d5d8e59a0ac05384be
3. Arch Linux kernel编译：https://wiki.archlinux.org/title/Kernel/Traditional_compilation
4. 参考的fix：https://bugzilla.kernel.org/attachment.cgi?id=303828&action=diff
5. 发布在Gists的个人补丁与社区讨论：https://gist.github.com/levihuayuzhang/6137ae4ae46301a355fd37c63e0d876a
## Eclipse CDT�°�װRust���

    ����:Cyper����ת����ע������.
    
����ֻ����о����Eclipse�û�
####1. ��װRust nightly  ���ԣ�
��װ��ɺ�ʹ������`which rustc`�����Կ�����װ·��Ϊ`/usr/local/bin/rustc`, �˴��İ�װλ��`/usr/local`������Ҫ����.

####2. ��װEclipse CDT���ԣ�
�������µ�CDT��ѹ

####3. ��װRustDT
�ο����http://rustdt.github.io/, ��Help > install new software������update site: https://rustdt.github.io/releases/, ��һ���������һ�£�������������Ҫ�ģ������ر��ᵽUsers in ChinaҲ������������������https://github.com/RustDT/rustdt.github.io/archive/master.zip
####4.����RustԴ��
������ҳ����, `rustc-nightly-src.tar.gz`, 20��M���ҽ����ѹ��/usr/local/srcĿ��.�����û�������
`export RUST_SRC_PATH=/usr/local/src/rustc-nightly/src`

####5.��װracer(Rust auto complete - er)

    git clone https://github.com/phildawes/racer.git
    cd racer; cargo build --release

�ⲽ������Ҫ�ȼ����ӣ�����������targetĿ¼�»������racer����.

####6. ����RustDT
ֱ�ӿ�ͼ����һ����rust�İ�װĿ¼���ڶ�����rustԴ��Ŀ¼����������racer��λ��.

![RustDT preference](../images/rustdt-pref.png)

####7. �Զ����ݼ�
���Ƚ�ԭʼ����jar���е�templates�ļ��������޸�:
`/home/cyper/.eclipse/org.eclipse.platform_4.4.2_1504408775_linux_gtk_x86_64/plugins/com.github.rustdt.ide.ui_0.2.0.v201504232210.jar/templates/default-templates.xml`

���jar��һ�������Eclipse��װĿ¼��pluginsĿ¼��

�޸�default-templates.xml���£�
```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<templates>
<template autoinsert="true" context="com.github.rustdt.ide.ui.TemplateContextType" deleted="false" id="BUG" description="BUG here comment" enabled="true" name="BUG">/* FIXME: BUG here${cursor}*/</template>
<template autoinsert="true" context="com.github.rustdt.ide.ui.TemplateContextType" deleted="false" id="FIXME" description="FIXME comment" enabled="true" name="FIXME">/* FIXME: ${cursor}*/</template>
<template autoinsert="true" context="com.github.rustdt.ide.ui.TemplateContextType" deleted="false" id="header_comment" description="header comment" enabled="true" name="headerbar">/* ----------------- ${header} ----------------- */</template>
<template autoinsert="true" context="com.github.rustdt.ide.ui.TemplateContextType" deleted="false" id="forin" description="iterate over an iterable" enabled="true" name="forin">for ${iterable_element} in ${iterable} {&#13;
	${cursor}&#13;
}</template>
<template autoinsert="true" context="com.github.rustdt.ide.ui.TemplateContextType" deleted="false" id="pr" description="println" enabled="true" name="pr">println!("${var}");</template>
<template autoinsert="true" context="com.github.rustdt.ide.ui.TemplateContextType" deleted="false" id="pl" description="println variable" enabled="true" name="pl">println!("{}", ${var});</template>
<template autoinsert="true" context="com.github.rustdt.ide.ui.TemplateContextType" deleted="false" id="pld" description="println debug variable" enabled="true" name="pld">println!("{:?}", ${var});</template>
</templates>
```
�ο������[pull request]����Ȼ�����Լ�������һ������pr��ģ�� **ֻҪ����pr��������println!("");���ҹ��ͣ��˫�����м�;**���棬����Eclipse.

����ͼ.

![RustDT ScreenShot](../images/rustdt-screenshot.jpg)

[pull request]: https://github.com/waynenilsen/RustDT/commit/16427af0c6f4569e9b5e0b154ad262832a080b52






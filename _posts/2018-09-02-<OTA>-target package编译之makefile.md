##### 1.1 编译target files依赖的目标

首先，列出编译target-files依赖的目标。如boot/system/vendor等image，imgdifff/bsdiff/releasetool等编译OTA包的工具。

```makefile
$(BUILT_TARGET_FILES_PACKAGE): \
        # boot image相关目标
        $(INSTALLED_BOOTIMAGE_TARGET) \
        # radio image相关目标
        $(INSTALLED_RADIOIMAGE_TARGET) \
        # recovery image相关目标
        $(INSTALLED_RECOVERYIMAGE_TARGET) \
        # system/user image依赖的文件
        $(FULL_SYSTEMIMAGE_DEPS) \
        # userdata image相关目标
        $(INSTALLED_USERDATAIMAGE_TARGET) \
        # cache image相关目标
        $(INSTALLED_CACHEIMAGE_TARGET) \
        # vendor image相关目标
        $(INSTALLED_VENDORIMAGE_TARGET) \
        # product image相关目标
        $(INSTALLED_PRODUCTIMAGE_TARGET) \
        # vbmeta image相关目标
        $(INSTALLED_VBMETAIMAGE_TARGET) \
        # dtbo image相关目标
        $(INSTALLED_DTBOIMAGE_TARGET) \
        # system image相关目标
        $(INTERNAL_SYSTEMOTHERIMAGE_FILES) \
        # android-info.txt
        $(INSTALLED_ANDROID_INFO_TXT_TARGET) \
        # kernel文件
        $(INSTALLED_KERNEL_TARGET) \
        # 2ndbootloader文件
        $(INSTALLED_2NDBOOTLOADER_TARGET) \
        # system base fs
        $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SYSTEM_BASE_FS_PATH) \
        # vendor base fs
        $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_VENDOR_BASE_FS_PATH) \
        # product base fs
        $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_PRODUCT_BASE_FS_PATH) \
        # selinux目录file_contexts.bin
        $(SELINUX_FC) \
        # apkcerts.txt
        $(APKCERTS_FILE) \
        # soong_zip文件
        $(SOONG_ZIP) \
        # fs_config工具
        $(HOST_OUT_EXECUTABLES)/fs_config \
        # imgdiff工具
        $(HOST_OUT_EXECUTABLES)/imgdiff \
        # bsdiff工具
        $(HOST_OUT_EXECUTABLES)/bsdiff \
        # build/make/tools/releasetools目录*.py脚本
        $(BUILD_IMAGE_SRCS) \
        # vendor_manifest.xml
        $(BUILT_VENDOR_MANIFEST) \
        # vendor_matrix.xml
        $(BUILT_VENDOR_MATRIX) \
        | $(ACP)
```

##### 1.2 创建target files目录

将上述依赖的目标列入target-files list，创建target-files目录，为后续打包作准备。

```makefile
    @echo "Package target files: $@"
    # system/vendor -> vendor链接
    $(call create-system-vendor-symlink)
    # system/product -> product链接
    $(call create-system-product-symlink)
    # 删除*.list文件
    $(hide) rm -rf $@ $@.list $(zip_root)
    # 创建target file package目录
    $(hide) mkdir -p $(dir $@) $(zip_root)
```

##### 1.3 recovery as boot处理

若有定义recovery as boot为true，则将recovery/root，2ndbootloader，kernel，dtbo，cmdline等内容拷贝至recovery目录，否则将其直接拷入boot目录。

```makefile
# 若BOARD_USES_RECOVERY_AS_BOOT为true，则处理recovery image相关目标
ifneq (,$(INSTALLED_RECOVERYIMAGE_TARGET)$(filter true,$(BOARD_USES_RECOVERY_AS_BOOT)))
    # PRIVATE_RECOVERY_OUT即创建RECOVERY目录
    $(hide) mkdir -p $(zip_root)/$(PRIVATE_RECOVERY_OUT)
    # 将recovery/root目录拷至RECOVERY/RAMDISK目录
    $(hide) $(call package_files-copy-root, \
        $(TARGET_RECOVERY_ROOT_OUT),$(zip_root)/$(PRIVATE_RECOVERY_OUT)/RAMDISK)
    # 若INSTALLED_KERNEL_TARGET为true,则将kernel拷贝至RECVOERY目录
    ifdef INSTALLED_KERNEL_TARGET
        $(hide) cp $(INSTALLED_KERNEL_TARGET) $(zip_root)/$(PRIVATE_RECOVERY_OUT)/kernel
    endif
    # 若有定义INSTALLED_2NDBOOTLOADER_TARGET，则将2ndbootloader拷至RECOVERY目录并重命名为second
    ifdef INSTALLED_2NDBOOTLOADER_TARGET
        $(hide) cp $(INSTALLED_2NDBOOTLOADER_TARGET) $(zip_root)/$(PRIVATE_RECOVERY_OUT)/second
    endif
    # 若有定义BOARD_INCLUDE_RECOVERY_DTBO，则将dtbo image目标拷至RECOVERY目录并重命名为recovery_dtbo
    ifdef BOARD_INCLUDE_RECOVERY_DTBO
        $(hide) cp $(INSTALLED_DTBOIMAGE_TARGET) $(zip_root)/$(PRIVATE_RECOVERY_OUT)/recovery_dtbo
    endif
    # 若有定义INTERNAL_KERNEL_CMDLINE，则将cmdline拷至RECOVERY目录
    ifdef INTERNAL_KERNEL_CMDLINE
        $(hide) echo "$(INTERNAL_KERNEL_CMDLINE)" > $(zip_root)/$(PRIVATE_RECOVERY_OUT)/cmdline
    endif
    # 若有定义BOARD_KERNEL_BASE，则将其写入RECOVERY目录base文件
    ifdef BOARD_KERNEL_BASE
        $(hide) echo "$(BOARD_KERNEL_BASE)" > $(zip_root)/$(PRIVATE_RECOVERY_OUT)/base
    endif
    # 若有定义BOARD_KERNEL_BASE，则将其写入RECOVERY目录pagesize文件
    ifdef BOARD_KERNEL_PAGESIZE
        $(hide) echo "$(BOARD_KERNEL_PAGESIZE)" > $(zip_root)/$(PRIVATE_RECOVERY_OUT)/pagesize
    endif
endif
    # 创建BOOT目录
    $(hide) mkdir -p $(zip_root)/BOOT
# 若BOARD_BUILD_SYSTEM_ROOT_IMAGE为true，则创建ROOT目录，并将root目录拷至ROOT目录，否则将root目录拷贝至BOOT/RAMDISK目录
ifeq ($(BOARD_BUILD_SYSTEM_ROOT_IMAGE),true)
    $(hide) mkdir -p $(zip_root)/ROOT
    $(hide) $(call package_files-copy-root, \
        $(TARGET_ROOT_OUT),$(zip_root)/ROOT)
else
    $(hide) $(call package_files-copy-root, \
        $(TARGET_ROOT_OUT),$(zip_root)/BOOT/RAMDISK)
endif
# 若BOARD_USES_RECOVERY_AS_BOOT为false，则将kernel拷贝至BOOT目录
ifneq ($(BOARD_USES_RECOVERY_AS_BOOT),true)
    ifdef INSTALLED_KERNEL_TARGET
        $(hide) cp $(INSTALLED_KERNEL_TARGET) $(zip_root)/BOOT/kernel
    endif
    # 若未定义INSTALLED_2NDBOOTLOADER_TARGET，则将2ndbootloader拷至BOOT目录并重命名为second
    ifdef INSTALLED_2NDBOOTLOADER_TARGET
        $(hide) cp $(INSTALLED_2NDBOOTLOADER_TARGET) $(zip_root)/BOOT/second
    endif
    # 若未定义INTERNAL_KERNEL_CMDLINE，则将cmdline拷至BOOT目录
    ifdef INTERNAL_KERNEL_CMDLINE
        $(hide) echo "$(INTERNAL_KERNEL_CMDLINE)" > $(zip_root)/BOOT/cmdline
    endif
    # 若未定义BOARD_KERNEL_BASE，则将base写入BOOT目录base文件
    ifdef BOARD_KERNEL_BASE
        $(hide) echo "$(BOARD_KERNEL_BASE)" > $(zip_root)/BOOT/base
    endif
    # 若未定义BOARD_KERNEL_PAGESIZE，则将base写入BOOT目录pagesize文件
    ifdef BOARD_KERNEL_PAGESIZE
        $(hide) echo "$(BOARD_KERNEL_PAGESIZE)" > $(zip_root)/BOOT/pagesize
    endif
endif
```

#### 1.4 radio/system/vendor等img处理

在target-files创建RADIO/VENDOR/SYSTEM等目录，并将out目录对应内容拷贝至此。

```makefile
    # 创建RADIO目录，并按radio image层次目录将radio image目标拷入
    $(hide) $(foreach t,$(INSTALLED_RADIOIMAGE_TARGET),\
                mkdir -p $(zip_root)/RADIO; \
                cp $(t) $(zip_root)/RADIO/$(notdir $(t));)
    # 将system image源文件拷入SYSTEM目录
    $(hide) $(call package_files-copy-root, \
        $(SYSTEMIMAGE_SOURCE_DIR),$(zip_root)/SYSTEM)
    # 将data目录拷入DATA目录
    $(hide) $(call package_files-copy-root, \
        $(TARGET_OUT_DATA),$(zip_root)/DATA)
# 若有定义BOARD_VENDORIMAGE_FILE_SYSTEM_TYPE，则将vendor目录拷入VENDOR目录
ifdef BOARD_VENDORIMAGE_FILE_SYSTEM_TYPE
    $(hide) $(call package_files-copy-root, \
        $(TARGET_OUT_VENDOR),$(zip_root)/VENDOR)
endif
# 若有定义BOARD_PRODUCTIMAGE_FILE_SYSTEM_TYPE，则将product目录拷入PRODUCT目录
ifdef BOARD_PRODUCTIMAGE_FILE_SYSTEM_TYPE
    $(hide) $(call package_files-copy-root, \
        $(TARGET_OUT_PRODUCT),$(zip_root)/PRODUCT)
endif
# 若有定义INSTALLED_SYSTEMOTHERIMAGE_TARGET，则将system_other目录拷入SYSTEM_OTHER目录
ifdef INSTALLED_SYSTEMOTHERIMAGE_TARGET
    $(hide) $(call package_files-copy-root, \
        $(TARGET_OUT_SYSTEM_OTHER),$(zip_root)/SYSTEM_OTHER)
endif
    # 创建OTA目标，并将android-info.txt拷入OTA目录
    $(hide) mkdir -p $(zip_root)/OTA
    $(hide) cp $(INSTALLED_ANDROID_INFO_TXT_TARGET) $(zip_root)/OTA/
# 若system base fs非空，则将其拷至META目录
ifneq ($(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SYSTEM_BASE_FS_PATH),)
    $(hide) cp $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SYSTEM_BASE_FS_PATH) \
      $(zip_root)/META/$(notdir $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_SYSTEM_BASE_FS_PATH))
endif
# 若vendor base fs非空，则将其拷至META目录
ifneq ($(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_VENDOR_BASE_FS_PATH),)
    $(hide) cp $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_VENDOR_BASE_FS_PATH) \
      $(zip_root)/META/$(notdir $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_VENDOR_BASE_FS_PATH))
endif
# 若product base fs非空，则将其拷至META目录
ifneq ($(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_PRODUCT_BASE_FS_PATH),)
    $(hide) cp $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_PRODUCT_BASE_FS_PATH) \
      $(zip_root)/META/$(notdir $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_PRODUCT_BASE_FS_PATH))
endif
```

##### 1.5 ota tool/apkcert/key/selinux处理处理

若为Non A/B项目，则将ota tool拷入OTA/bin目录。
将apkcerts/tool/otakeys/file_contexts.bin拷入META目录。

```makefile
# 若AB_OTA_UPDATER为false，则创建OTA/bin目录，并将updater拷入OTA/bin目录
ifneq ($(AB_OTA_UPDATER),true)
ifneq ($(built_ota_tools),)
    $(hide) mkdir -p $(zip_root)/OTA/bin
    $(hide) cp $(PRIVATE_OTA_TOOLS) $(zip_root)/OTA/bin/
endif
endif
    # 创建META目录，并将apkcerts.txt拷入META目录
    $(hide) mkdir -p $(zip_root)/META
    $(hide) cp $(APKCERTS_FILE) $(zip_root)/META/apkcerts.txt
# 若tool_extension非空，则将releasetools拷入META目录
ifneq ($(tool_extension),)
    $(hide) cp $(PRIVATE_TOOL_EXTENSION) $(zip_root)/META/
endif
    # 将release key写入META目录otakeys.txt文件
    $(hide) echo "$(PRODUCT_OTA_PUBLIC_KEYS)" > $(zip_root)/META/otakeys.txt
    # 将selinux目录file_contexts.bin文件拷入META目录
    $(hide) cp $(SELINUX_FC) $(zip_root)/META/file_contexts.bin
```

##### 1.6 生成misc_info.txt文件

```makefile
    # 将recovery api版本信息赋给recovery_api_version并写入META目录misc_info.txt文件
    $(hide) echo "recovery_api_version=$(PRIVATE_RECOVERY_API_VERSION)" > $(zip_root)/META/misc_info.txt
    # 将fstab版本信息赋给fstab_version并写入META目录misc_info.txt文件
    $(hide) echo "fstab_version=$(PRIVATE_RECOVERY_FSTAB_VERSION)" >> $(zip_root)/META/misc_info.txt
# 若有定义BOARD_FLASH_BLOCK_SIZE，则将其赋给blocksize并写入META目录misc_info.txt文件
ifdef BOARD_FLASH_BLOCK_SIZE
    $(hide) echo "blocksize=$(BOARD_FLASH_BLOCK_SIZE)" >> $(zip_root)/META/misc_info.txt
endif
# 若有定义BOARD_BOOTIMAGE_PARTITION_SIZE，则将其赋给boot_siz并写入META目录misc_info.txt文件
ifdef BOARD_BOOTIMAGE_PARTITION_SIZE
    $(hide) echo "boot_size=$(BOARD_BOOTIMAGE_PARTITION_SIZE)" >> $(zip_root)/META/misc_info.txt
endif
# 若INSTALLED_RECOVERYIMAGE_TARGET为空，则将no_recovery=true写入META目录misc_info.txt文件
ifeq ($(INSTALLED_RECOVERYIMAGE_TARGET),)
    $(hide) echo "no_recovery=true" >> $(zip_root)/META/misc_info.txt
endif
# 若有定义BOARD_INCLUDE_RECOVERY_DTBO，则将include_recovery_dtbo=true写入META目录misc_info.txt文件
ifdef BOARD_INCLUDE_RECOVERY_DTBO
    $(hide) echo "include_recovery_dtbo=true" >> $(zip_root)/META/misc_info.txt
endif
# 若有定义BOARD_RECOVERYIMAGE_PARTITION_SIZE，则将其赋给recovery_size并写入META目录misc_info.txt文件
ifdef BOARD_RECOVERYIMAGE_PARTITION_SIZE
    $(hide) echo "recovery_size=$(BOARD_RECOVERYIMAGE_PARTITION_SIZE)" >> $(zip_root)/META/misc_info.txt
endif
# 若有定义TARGET_RECOVERY_FSTYPE_MOUNT_OPTIONS，则将其赋给recovery_mount_options，否则将DEFAULT_TARGET_RECOVERY_FSTYPE_MOUNT_OPTIONS赋给recovery_mount_options，并写入META目录misc_info.txt文件
ifdef TARGET_RECOVERY_FSTYPE_MOUNT_OPTIONS
    @# TARGET_RECOVERY_FSTYPE_MOUNT_OPTIONS can be empty to indicate that nothing but defaults should be used.
    $(hide) echo "recovery_mount_options=$(TARGET_RECOVERY_FSTYPE_MOUNT_OPTIONS)" >> $(zip_root)/META/misc_info.txt
else
    $(hide) echo "recovery_mount_options=$(DEFAULT_TARGET_RECOVERY_FSTYPE_MOUNT_OPTIONS)" >> $(zip_root)/META/misc_info.txt
endif
    # 将PRIVATE_TOOL_EXTENSIONS赋给tool_extensions，并写入META目录misc_info.txt文件
    $(hide) echo "tool_extensions=$(PRIVATE_TOOL_EXTENSIONS)" >> $(zip_root)/META/misc_info.txt
    # 将DEFAULT_SYSTEM_DEV_CERTIFICATE赋给default_system_dev_certificate，并写入META目录misc_info.txt文件
    $(hide) echo "default_system_dev_certificate=$(DEFAULT_SYSTEM_DEV_CERTIFICATE)" >> $(zip_root)/META/misc_info.txt
# 若有定义PRODUCT_EXTRA_RECOVERY_KEYS，则将其赋给extra_recovery_keys，并写入META目录misc_info.txt文件
ifdef PRODUCT_EXTRA_RECOVERY_KEYS
    $(hide) echo "extra_recovery_keys=$(PRODUCT_EXTRA_RECOVERY_KEYS)" >> $(zip_root)/META/misc_info.txt
endif
    # 将BOARD_MKBOOTIMG_ARGS赋给mkbootimg_args，并写入META目录misc_info.txt文件
    $(hide) echo 'mkbootimg_args=$(BOARD_MKBOOTIMG_ARGS)' >> $(zip_root)/META/misc_info.txt
    # 将INTERNAL_MKBOOTIMG_VERSION_ARGS赋给mkbootimg_version_args，并写入META目录misc_info.txt文件
    $(hide) echo 'mkbootimg_version_args=$(INTERNAL_MKBOOTIMG_VERSION_ARGS)' >> $(zip_root)/META/misc_info.txt
    # 将multistage_support=1写入META目录misc_info.txt文件
    $(hide) echo "multistage_support=1" >> $(zip_root)/META/misc_info.txt
    # 将Dblockimgdiff_versions=3,4写入META目录misc_info.txt文件
    $(hide) echo "blockimgdiff_versions=3,4" >> $(zip_root)/META/misc_info.txt
# 若OEM_THUMBPRINT_PROPERTIES非空，则将其赋给oem_fingerprint_properties，，并写入META目录misc_info.txt文件
ifneq ($(OEM_THUMBPRINT_PROPERTIES),)
    $(hide) echo "oem_fingerprint_properties=$(OEM_THUMBPRINT_PROPERTIES)" >> $(zip_root)/META/misc_info.txt
endif
# 若SANITIZE_TARGET非空，则将userdata_img_with_data=true写入META目录misc_info.txt文件
ifneq ($(strip $(SANITIZE_TARGET)),)
    $(hide) echo "userdata_img_with_data=true" >> $(zip_root)/META/misc_info.txt
endif
# 若BOARD_USES_FULL_RECOVERY_IMAGE为true，则将full_recovery_image=true写入META目录misc_info.txt文件
ifeq ($(BOARD_USES_FULL_RECOVERY_IMAGE),true)
    $(hide) echo "full_recovery_image=true" >> $(zip_root)/META/misc_info.txt
endif
# 若有定义BOARD_BPT_INPUT_FILES，则：
# 将board_bpt_enable=true写入META目录misc_info.txt文件
# 将BOARD_BPT_MAKE_TABLE_ARGS赋给board_bpt_make_table_args，并写入META目录misc_info.txt文件
# 将BOARD_BPT_INPUT_FILES赋给board_bpt_input_files，并写入META目录misc_info.txt文件
ifdef BOARD_BPT_INPUT_FILES
    $(hide) echo "board_bpt_enable=true" >> $(zip_root)/META/misc_info.txt
    $(hide) echo "board_bpt_make_table_args=$(BOARD_BPT_MAKE_TABLE_ARGS)" >> $(zip_root)/META/misc_info.txt
    $(hide) echo "board_bpt_input_files=$(BOARD_BPT_INPUT_FILES)" >> $(zip_root)/META/misc_info.txt
endif
# 若有定义BOARD_BPT_DISK_SIZE，则将其赋给board_bpt_disk_size，并写入META目录misc_info.txt文件
ifdef BOARD_BPT_DISK_SIZE
    $(hide) echo "board_bpt_disk_size=$(BOARD_BPT_DISK_SIZE)" >> $(zip_root)/META/misc_info.txt
endif
    # 将userimage prop写入META目录misc_info.txt文件
    $(call generate-userimage-prop-dictionary, $(zip_root)/META/misc_info.txt)
```

##### 1.7 将AVB信息到misc_info中

```makefile
# 若BOARD_AVB_ENABLE为true，则：
# 将avb_enable=true"写入META目录misc_info.txt文件
# 将BOARD_AVB_KEY_PATH赋给avb_vbmeta_key_path，并写入META目录misc_info.txt文件
# 将BOARD_AVB_ALGORITHM赋给avb_vbmeta_algorithm，并写入META目录misc_info.txt文件
# 将BOARD_AVB_MAKE_VBMETA_IMAGE_ARGS赋给avb_vbmeta_args，并写入META目录misc_info.txt文件
# 将BOARD_AVB_BOOT_ADD_HASH_FOOTER_ARGS赋给avb_boot_add_hash_footer_args，并写入META目录misc_info.txt文件
ifeq ($(BOARD_AVB_ENABLE),true)
    $(hide) echo "avb_enable=true" >> $(zip_root)/META/misc_info.txt
    $(hide) echo "avb_vbmeta_key_path=$(BOARD_AVB_KEY_PATH)" >> $(zip_root)/META/misc_info.txt
    $(hide) echo "avb_vbmeta_algorithm=$(BOARD_AVB_ALGORITHM)" >> $(zip_root)/META/misc_info.txt
    $(hide) echo "avb_vbmeta_args=$(BOARD_AVB_MAKE_VBMETA_IMAGE_ARGS)" >> $(zip_root)/META/misc_info.txt
    $(hide) echo "avb_boot_add_hash_footer_args=$(BOARD_AVB_BOOT_ADD_HASH_FOOTER_ARGS)" >> $(zip_root)/META/misc_info.txt
    # 若有定义BOARD_AVB_BOOT_KEY_PATH，则：
    # 将BOARD_AVB_BOOT_KEY_PATH赋给avb_boot_key_path，并写入META目录misc_info.txt文件
    # 将BOARD_AVB_BOOT_ALGORITHM赋给avb_boot_algorithm，并写入META目录misc_info.txt文件
    # 将BOARD_AVB_BOOT_ROLLBACK_INDEX_LOCATION赋给avb_boot_rollback_index_location，并写入META目录misc_info.txt文件
ifdef BOARD_AVB_BOOT_KEY_PATH
    $(hide) echo "avb_boot_key_path=$(BOARD_AVB_BOOT_KEY_PATH)" >> $(zip_root)/META/misc_info.txt
    $(hide) echo "avb_boot_algorithm=$(BOARD_AVB_BOOT_ALGORITHM)" >> $(zip_root)/META/misc_info.txt
    $(hide) echo "avb_boot_rollback_index_location=$(BOARD_AVB_BOOT_ROLLBACK_INDEX_LOCATION)" >> $(zip_root)/META/misc_info.txt
endif
    # 将BOARD_AVB_RECOVERY_ADD_HASH_FOOTER_ARGS赋给avb_recovery_add_hash_footer_args，并写入META目录misc_info.txt文件
    $(hide) echo "avb_recovery_add_hash_footer_args=$(BOARD_AVB_RECOVERY_ADD_HASH_FOOTER_ARGS)" >> $(zip_root)/META/misc_info.txt
    # 若有定义BOARD_AVB_RECOVERY_KEY_PATH，则：
    # 将BOARD_AVB_RECOVERY_KEY_PATH赋给avb_recovery_key_path，并写入META目录misc_info.txt文件
    # 将BOARD_AVB_RECOVERY_ALGORITHM赋给avb_recovery_algorithm，并写入META目录misc_info.txt文件
    # 将BOARD_AVB_RECOVERY_ROLLBACK_INDEX_LOCATION赋给avb_recovery_rollback_index_location，并写入META目录misc_info.txt文件
ifdef BOARD_AVB_RECOVERY_KEY_PATH
    $(hide) echo "avb_recovery_key_path=$(BOARD_AVB_RECOVERY_KEY_PATH)" >> $(zip_root)/META/misc_info.txt
    $(hide) echo "avb_recovery_algorithm=$(BOARD_AVB_RECOVERY_ALGORITHM)" >> $(zip_root)/META/misc_info.txt
    $(hide) echo "avb_recovery_rollback_index_location=$(BOARD_AVB_RECOVERY_ROLLBACK_INDEX_LOCATION)" >> $(zip_root)/META/misc_info.txt
endif
endif
```

##### 1.7 将A/B Updater信息写入misc_info

```makefile
# AB_OTA_UPDATER为true
ifeq ($(AB_OTA_UPDATER),true)
    # 将system/update_engine/update_engine.conf拷入META目录update_engine_config.txt文件
    $(hide) cp $(TOPDIR)system/update_engine/update_engine.conf $(zip_root)/META/update_engine_config.txt
    # 将AB_OTA_PARTITIONS定义的内容写入META目录ab_partitions.txt文件
    $(hide) for part in $(AB_OTA_PARTITIONS); do \
      echo "$${part}" >> $(zip_root)/META/ab_partitions.txt; \
    done
    # 将AB_OTA_POSTINSTALL_CONFIG配置的值写入META目录postinstall_config.txt文件
    $(hide) for conf in $(AB_OTA_POSTINSTALL_CONFIG); do \
      echo "$${conf}" >> $(zip_root)/META/postinstall_config.txt; \
    done
    # 将TARGET_BUILD_VARIANT赋给build_type，并写入META目录misc_info文件
    $(hide) echo "build_type=$(TARGET_BUILD_VARIANT)" >> $(zip_root)/META/misc_info.txt
    # 将ab_update=true写入META目录misc_info文件
    $(hide) echo "ab_update=true" >> $(zip_root)/META/misc_info.txt
    # 若有定义BRILLO_VENDOR_PARTITIONS，则创建VENDOR_IMAGE目录，并将其内容拷入
    ifdef BRILLO_VENDOR_PARTITIONS
        $(hide) mkdir -p $(zip_root)/VENDOR_IMAGES
        $(hide) for f in $(BRILLO_VENDOR_PARTITIONS); do \
        pair1="$$(echo $$f | awk -F':' '{print $$1}')"; \
        pair2="$$(echo $$f | awk -F':' '{print $$2}')"; \
        src=$${pair1}/$${pair2}; \
        dest=$(zip_root)/VENDOR_IMAGES/$${pair2}; \
        mkdir -p $$(dirname "$${dest}"); \
        cp $${src} $${dest}; \
        done;
    endif
    # 若有定义OSRELEASED_DIRECTORY，则将product_id，product_version，system_version拷贝至META目录
    ifdef OSRELEASED_DIRECTORY
        $(hide) cp $(TARGET_OUT_OEM)/$(OSRELEASED_DIRECTORY)/product_id $(zip_root)/META/product_id.txt
        $(hide) cp $(TARGET_OUT_OEM)/$(OSRELEASED_DIRECTORY)/product_version $(zip_root)/META/product_version.txt
        $(hide) cp $(TARGET_OUT_ETC)/$(OSRELEASED_DIRECTORY)/system_version $(zip_root)/META/system_version.txt
    endif
endif
```

##### 1.8 处理recovery及预编译的vendor/boot/dtbo等imag

```makefile
# 若INSTALLED_RECOVERYIMAGE_TARGET非空，则调用make_recovery_patch脚本
ifneq ($(INSTALLED_RECOVERYIMAGE_TARGET),)
    $(hide) PATH=$(foreach p,$(INTERNAL_USERIMAGES_BINARY_PATHS),$(p):)$$PATH MKBOOTIMG=$(MKBOOTIMG) \
        build/make/tools/releasetools/make_recovery_patch $(zip_root) $(zip_root)
endif
# 若BREAKPAD_GENERATE_SYMBOLS为true，则将breakpad加入BREAKPAD目录
ifeq ($(BREAKPAD_GENERATE_SYMBOLS),true)
    $(hide) $(ACP) -r $(TARGET_OUT_BREAKPAD) $(zip_root)/BREAKPAD
endif
# 若有定义BOARD_BUILD_DISABLED_VBMETAIMAGE，则将vbmeta image目标拷入IMAGES目录
ifeq (true,$(BOARD_BUILD_DISABLED_VBMETAIMAGE))
    $(hide) mkdir -p $(zip_root)/IMAGES
    $(hide) cp $(INSTALLED_VBMETAIMAGE_TARGET) $(zip_root)/IMAGES/
endif
# 若有定义BOARD_PREBUILT_VENDORIMAGE，则将vendor image目标拷入IMAGES目录
ifdef BOARD_PREBUILT_VENDORIMAGE
    $(hide) mkdir -p $(zip_root)/IMAGES
    $(hide) cp $(INSTALLED_VENDORIMAGE_TARGET) $(zip_root)/IMAGES/
endif
# 若有定义BOARD_PREBUILT_PRODUCTIMAGE，则将product image目标拷入IMAGES目录
ifdef BOARD_PREBUILT_PRODUCTIMAGE
    $(hide) mkdir -p $(zip_root)/IMAGES
    $(hide) cp $(INSTALLED_PRODUCTIMAGE_TARGET) $(zip_root)/IMAGES/
endif
# 若有定义BOARD_PREBUILT_BOOTIMAGE，则将boot image目标拷入IMAGES目录
ifdef BOARD_PREBUILT_BOOTIMAGE
    $(hide) mkdir -p $(zip_root)/IMAGES
    $(hide) cp $(INSTALLED_BOOTIMAGE_TARGET) $(zip_root)/IMAGES/
endif
# 若有定义BOARD_PREBUILT_DTBOIMAGE，则创建PREBUILT_IMAGES目录，并将dtbo image拷入PREBUILT_IMAGES目录
ifdef BOARD_PREBUILT_DTBOIMAGE
    $(hide) mkdir -p $(zip_root)/PREBUILT_IMAGES
    $(hide) cp $(INSTALLED_DTBOIMAGE_TARGET) $(zip_root)/PREBUILT_IMAGES/
    # 将has_dtbo=true写入META目录misc_info.txt文件
    $(hide) echo "has_dtbo=true" >> $(zip_root)/META/misc_info.txt
    # 若BOARD_AVB_ENABLE为true,则：
    # 将BOARD_DTBOIMG_PARTITION_SIZE赋给dtbo_size，并写入META目录misc_info.txt文件
    # 将BOARD_AVB_DTBO_ADD_HASH_FOOTER_ARGS赋给avb_dtbo_add_hash_footer_args，并写入META目录misc_info.txt文件
    ifeq ($(BOARD_AVB_ENABLE),true)
        $(hide) echo "dtbo_size=$(BOARD_DTBOIMG_PARTITION_SIZE)" >> $(zip_root)/META/misc_info.txt
        $(hide) echo "avb_dtbo_add_hash_footer_args=$(BOARD_AVB_DTBO_ADD_HASH_FOOTER_ARGS)" >> $(zip_root)/META/misc_info.txt
        # 若有定义BOARD_AVB_DTBO_KEY_PATH，则：
        # 将BOARD_AVB_DTBO_KEY_PATH赋给avb_dtbo_key_path，并写入META目录misc_info.txt文件
        # 将BOARD_AVB_DTBO_ALGORITHM赋给avb_dtbo_algorithm，并写入META目录misc_info.txt文件
        # 将BOARD_AVB_DTBO_ROLLBACK_INDEX_LOCATION赋给avb_dtbo_rollback_index_location，并写入META目录misc_info.txt文件
        ifdef BOARD_AVB_DTBO_KEY_PATH
            $(hide) echo "avb_dtbo_key_path=$(BOARD_AVB_DTBO_KEY_PATH)" >> $(zip_root)/META/misc_info.txt
            $(hide) echo "avb_dtbo_algorithm=$(BOARD_AVB_DTBO_ALGORITHM)" >> $(zip_root)/META/misc_info.txt
            $(hide) echo "avb_dtbo_rollback_index_location=$(BOARD_AVB_DTBO_ROLLBACK_INDEX_LOCATION)" \
                >> $(zip_root)/META/misc_info.txt
        endif
    endif
endif
```

##### 1.9 处理radio/vendor/system等img的file config内容

```makefile
    # 将radio image信息写入META目录pack_radioimages.txt文件
    $(hide) $(foreach part,$(BOARD_PACK_RADIOIMAGES), \
        echo $(part) >> $(zip_root)/META/pack_radioimages.txt;)
    # 将则将运行fs_config，将system相关config写入META目录filesystem_config.txt文件
    $(hide) $(call fs_config,$(zip_root)/SYSTEM,system/) > $(zip_root)/META/filesystem_config.txt
# 若有定义BOARD_VENDORIMAGE_FILE_SYSTEM_TYPE，则将运行fs_config，将vendor相关config写入META目录vendor_filesystem_config.txt文件
ifdef BOARD_VENDORIMAGE_FILE_SYSTEM_TYPE
    $(hide) $(call fs_config,$(zip_root)/VENDOR,vendor/) > $(zip_root)/META/vendor_filesystem_config.txt
endif
# 若有定义BOARD_PRODUCTIMAGE_FILE_SYSTEM_TYPE，则将运行fs_config，将product相关config写入META目录product_filesystem_config.txt文件
ifdef BOARD_PRODUCTIMAGE_FILE_SYSTEM_TYPE
    $(hide) $(call fs_config,$(zip_root)/PRODUCT,product/) > $(zip_root)/META/product_filesystem_config.txt
endif
# 若BOARD_BUILD_SYSTEM_ROOT_IMAGE为true，则运行fs_config，将root相关config写入META目录root_filesystem_config.txt文件,否则将ramdisk写入META目录boot_filesystem_config.txt文件
ifeq ($(BOARD_BUILD_SYSTEM_ROOT_IMAGE),true)
    $(hide) $(call fs_config,$(zip_root)/ROOT,) > $(zip_root)/META/root_filesystem_config.txt
    # 若BOARD_USES_RECOVERY_AS_BOOT为true，则运行fs_config，将boot_ramdisk相关config写入META目录boot_filesystem_config.txt文件
    ifeq ($(BOARD_USES_RECOVERY_AS_BOOT),true)
        $(hide) $(call fs_config,$(zip_root)/BOOT/RAMDISK,) > $(zip_root)/META/boot_filesystem_config.txt
    endif
else
    $(hide) $(call fs_config,$(zip_root)/BOOT/RAMDISK,) > $(zip_root)/META/boot_filesystem_config.txt
endif
# 若INSTALLED_RECOVERYIMAGE_TARGET非空，则运行fs_config，将recovery_ramdisk相关config写入META目录recovery_filesystem_config.txt文件
ifneq ($(INSTALLED_RECOVERYIMAGE_TARGET),)
    $(hide) $(call fs_config,$(zip_root)/RECOVERY/RAMDISK,) > $(zip_root)/META/recovery_filesystem_config.txt
endif
# 若有定义INSTALLED_SYSTEMOTHERIMAGE_TARGET，则运行fs_config，将system_other相关config写入META目录system_other_filesystem_config.txt文件
ifdef INSTALLED_SYSTEMOTHERIMAGE_TARGET
    $(hide) $(call fs_config,$(zip_root)/SYSTEM_OTHER,system/) > $(zip_root)/META/system_other_filesystem_config.txt
endif
    # 拷贝system_manifest.xml，system_matrix.xml至META目录
    $(hide) cp $(BUILT_SYSTEM_MANIFEST) $(zip_root)/META/system_manifest.xml
    $(hide) cp $(BUILT_SYSTEM_COMPATIBILITY_MATRIX) $(zip_root)/META/system_matrix.xml
# 若有定义BUILT_VENDOR_MANIFEST，则将vendor_manifest.xml拷至META目录
ifdef BUILT_VENDOR_MANIFEST
    $(hide) cp $(BUILT_VENDOR_MANIFEST) $(zip_root)/META/vendor_manifest.xml
endif
# 若有定义BUILT_VENDOR_MATRIX，则将vendor_matrix.xml拷至META目录
ifdef BUILT_VENDOR_MATRIX
    $(hide) cp $(BUILT_VENDOR_MATRIX) $(zip_root)/META/vendor_matrix.xml
endif
    # 调用add_img_to_target_files脚本，将诸image拷入IMAGES/RADIO等目录
    $(hide) PATH=$(foreach p,$(INTERNAL_USERIMAGES_BINARY_PATHS),$(p):)$$PATH MKBOOTIMG=$(MKBOOTIMG) \
        build/make/tools/releasetools/add_img_to_target_files -a -v -p $(HOST_OUT) $(zip_root)
    # 查找META目录并按顺序输出至*.list文件中
    $(hide) find $(zip_root)/META | sort >$@.list
    $(hide) find $(zip_root) -path $(zip_root)/META -prune -o -print | sort >>$@.list
    # 按*.list文件进行target file package的打包
    $(hide) $(SOONG_ZIP) -d -o $@ -C $(zip_root) -l $@.list
```

##### 1.10 定义target-files-package模块编译

```makefile
# 用于make target-files-package
.PHONY: target-files-package
target-files-package: $(BUILT_TARGET_FILES_PACKAGE)
# 检测到编译指令包含，则执行BUILD target相关操作
ifneq ($(filter $(MAKECMDGOALS),target-files-package),)
$(call dist-for-goals, target-files-package, $(BUILT_TARGET_FILES_PACKAGE))
endift-for-goals, target-files-package, $(BUILT_TARGET_FILES_PACKAGE))
endif
```
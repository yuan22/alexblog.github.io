#### 一、编译指令

##### 1.1 差分包制作指令

```
ota_from_target_files --block -k release -i source_target_files.zip target_target_file.zip ota_package.zip
```

##### 1.2 整包制作指令

```
ota_from_target_files --block -k release input_target_files.zip full_ota_package.zip
```

#### 二、指令详解

##### 2.1 参数解释：

&emsp;-k (--package_key) <key>

指定OTA包签名文件。Android默认为build/target/product/security/testkey。实际操作中，不同项目会使用不同的releasekey。

&emsp;-i (--incremental_from) <file>

指定源版本target_file文件。有该参数则是编译差分包，同理无此参数即编译整包。

&emsp;--full_radio

编译差分包时用于限定完全升级radio。

&emsp;--full_bootloader

编译差分包时用于限定完全升级booloader。

&emsp;--verify

编译Non-A/B差分包时用于限定是否对system/vender分区verify。

&emsp;--wipe_user_data

用于指定升级时指定恢复出厂操作。

&emsp;--downgrade

允许版本向旧版本“升级”（降级），会执行恢复出厂操作。

&emsp;-2 (--two_step)

用于指定制作两步式升级的OTA包。先更新recovery分区再重启通过新recovery分区升级system分区。

&emsp;--block

用于指定non-A/B版本基于块方式编译OTA包，相比较文件方式更有优势。

&emsp;-t (--worker_threads) <int>

用于指定patch方式增量升级的工作线程数，默认为3。

&emsp;--stash_threshold <float>

用于指定缓存区最大阀值，默认为0.8。

&emsp;--log_diff <file>

用于指定以log文件形式保存制作差分包的diff信息。


##### 2.2 指令代码解释

```python
def main(argv):

  def option_handler(o, a):
    if o in ("-k", "--package_key"):
      OPTIONS.package_key = a
    elif o in ("-i", "--incremental_from"):
      OPTIONS.incremental_source = a
    elif o == "--full_radio":
      OPTIONS.full_radio = True
    elif o == "--full_bootloader":
      OPTIONS.full_bootloader = True
    elif o == "--wipe_user_data":
      OPTIONS.wipe_user_data = True
    elif o == "--downgrade":
      OPTIONS.downgrade = True
      OPTIONS.wipe_user_data = True
    ...
    elif o in ("-t", "--worker_threads"):
      if a.isdigit():
        OPTIONS.worker_threads = int(a)
      else:
        raise ValueError("Cannot parse value %r for option %r - only "
                         "integers are allowed." % (a, o))
    elif o in ("-2", "--two_step"):
      OPTIONS.two_step = True
    ...
    elif o == "--verify":
      OPTIONS.verify = True
    elif o == "--block":
      OPTIONS.block_based = True
    ...
    elif o == "--stash_threshold":
      try:
        OPTIONS.stash_threshold = float(a)
      except ValueError:
        raise ValueError("Cannot parse value %r for option %r - expecting "
                         "a float" % (a, o))
    elif o == "--log_diff":
      OPTIONS.log_diff = a
    ...
    else:
      return False
    return True

  args = common.ParseOptions(argv, __doc__,
                             extra_opts="b:k:i:d:e:t:2o:",
                             extra_long_opts=[
                                 "package_key=",
                                 "incremental_from=",
                                 "full_radio",
                                 "full_bootloader",
                                 "wipe_user_data",
                                 "downgrade",
                                 ...
                                 "two_step",
                                 ...
                                 "block",
                                 ...
                                 "verify",
                                 "stash_threshold=",
                                 "log_diff=",
                                ...
                             ], extra_option_handler=option_handler)

  if len(args) != 2:
    common.Usage(__doc__)
    sys.exit(1)

  if OPTIONS.downgrade:
    # 判断是否强制执行恢复出厂操作
    if not OPTIONS.wipe_user_data:
      raise ValueError("Cannot downgrade without a data wipe")

    # 允许降级
    if OPTIONS.incremental_source is None:
      raise ValueError("Cannot generate downgradable full OTAs")

  assert not (OPTIONS.downgrade and OPTIONS.timestamp), \
      "Cannot have --downgrade AND --override_timestamp both"

  ...

  ab_update = OPTIONS.info_dict.get("ab_update") == "true"

  # 指定签名文件
  if not OPTIONS.no_signing or ab_update:
    if OPTIONS.package_key is None:
      OPTIONS.package_key = OPTIONS.info_dict.get(
          "default_system_dev_certificate",
          "build/target/product/security/testkey")
    OPTIONS.key_passwords = common.GetKeyPasswords([OPTIONS.package_key])

  if ab_update:
    WriteABOTAPackageWithBrilloScript(
        target_file=args[0],
        output_file=args[1],
        source_file=OPTIONS.incremental_source)

    print("done.")
    return

  ...

  else:
    print("unzipping target target-files...")
    OPTIONS.input_tmp = common.UnzipTemp(args[0], UNZIP_PATTERN)
  OPTIONS.target_tmp = OPTIONS.input_tmp

  # 制作OTA包的工具
  if OPTIONS.device_specific is None:
    from_input = os.path.join(OPTIONS.input_tmp, "META", "releasetools.py")
    if os.path.exists(from_input):
      print("(using device-specific extensions from target_files)")
      OPTIONS.device_specific = from_input
    else:
      OPTIONS.device_specific = OPTIONS.info_dict.get("tool_extensions")

  if OPTIONS.device_specific is not None:
    OPTIONS.device_specific = os.path.abspath(OPTIONS.device_specific)

  # 设置生成的OTA包
  if OPTIONS.no_signing:
    if os.path.exists(args[1]):
      os.unlink(args[1])
    output_zip = zipfile.ZipFile(args[1], "w",
                                 compression=zipfile.ZIP_DEFLATED)
  else:
    temp_zip_file = tempfile.NamedTemporaryFile()
    output_zip = zipfile.ZipFile(temp_zip_file, "w",
                                 compression=zipfile.ZIP_DEFLATED)

  # 制作整包
  if OPTIONS.incremental_source is None:
    with zipfile.ZipFile(args[0], 'r') as input_zip:
      WriteFullOTAPackage(input_zip, output_zip)

  # 制作差分包
  else:
    print("unzipping source target-files...")
    OPTIONS.source_tmp = common.UnzipTemp(
        OPTIONS.incremental_source, UNZIP_PATTERN)
    with zipfile.ZipFile(args[0], 'r') as input_zip, \
        zipfile.ZipFile(OPTIONS.incremental_source, 'r') as source_zip:
      WriteBlockIncrementalOTAPackage(input_zip, source_zip, output_zip)

    # 保存diff log
    if OPTIONS.log_diff:
      with open(OPTIONS.log_diff, 'w') as out_file:
        import target_files_diff
        target_files_diff.recursiveDiff(
            '', OPTIONS.source_tmp, OPTIONS.input_tmp, out_file)

  common.ZipClose(output_zip)

  # 判断是否需要签名
  if not OPTIONS.no_signing:
    SignOutput(temp_zip_file.name, args[1])
    temp_zip_file.close()

  print("done.")
```

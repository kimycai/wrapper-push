# wrapper-push，虚线以下为githu baction脚本，存放于源仓库WorldObservation-wrapper/.github/workflows/*.yml文件中，*为自定义名称
----------------------------------------------------
name: Build

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-22.04
    steps:
    - uses: actions/checkout@v4

    - name: Install aria2
      run: sudo apt install aria2 -y

    - name: Install LLVM
      run: sudo bash -c "$(wget -O - https://apt.llvm.org/llvm.sh)"
    
    - name: Set up Android NDK r23b
      run: | 
        aria2c -o android-ndk-r23b-linux.zip https://dl.google.com/android/repository/android-ndk-r23b-linux.zip
        unzip -q -d ~ android-ndk-r23b-linux.zip
      
    - name: Build
      run: |
        mkdir build
        cd build
        cmake ..
        make

    - name: Set outputs
      id: vars
      run: echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

    - name: Upload artifact
      uses: actions/upload-artifact@v4
      with:
        name: Wrapper.x86_64.${{ steps.vars.outputs.sha_short }}
        include-hidden-files: true
        path: |
          rootfs
          wrapper
          Dockerfile

    - name: Create Release Zip File
      run: zip -r Wrapper.x86_64.${{ steps.vars.outputs.sha_short }}.zip rootfs/ wrapper Dockerfile
      
    - name: Create Release
      uses: softprops/action-gh-release@v2
      with:
        files: Wrapper.x86_64.${{ steps.vars.outputs.sha_short }}.zip
        body: ${{ github.event.head_commit.message }}
        name: Wrapper.x86_64.${{ steps.vars.outputs.sha_short }}
        tag_name: Wrapper.x86_64.${{ steps.vars.outputs.sha_short }}
        make_latest: true

    # ---------------------- 新增核心步骤：解压Release包并推送到wrapper-push ----------------------
    - name: Unzip Release package (get raw files)
      run: |
        # 创建临时解压目录，避免文件冲突
        mkdir -p temp-unzip
        # 解压Release压缩包到临时目录，保留原始文件/目录结构
        unzip -q Wrapper.x86_64.${{ steps.vars.outputs.sha_short }}.zip -d temp-unzip/
        # 验证解压结果，列出所有原生文件（方便排查）
        ls -lhR temp-unzip/

    - name: Configure Git identity (for push)
      run: |
        # 配置Git用户名和邮箱，Actions推送仓库必备
        git config --global user.name "GitHub Actions"
        git config --global user.email "actions@github.com"

    - name: Checkout wrapper-push repository
      uses: actions/checkout@v4
      with:
        repository: kimycai/wrapper-push  # 你的「用户名/wrapper-push」，固定
        ref: main                          # 推送到wrapper-push的main分支
        token: ${{ secrets.WRAPPER_PUSH_TOKEN }}  # 已配置的专属授权令牌
        path: wrapper-push                 # 隔离工作目录，与原项目无冲突
        fetch-depth: 1

    - name: Copy unzipped files & push to wrapper-push
      run: |
        # 递归复制解压后的所有原生文件（rootfs/、wrapper、Dockerfile），-p保留权限/时间戳
        cp -rp temp-unzip/* wrapper-push/
        # 进入wrapper-push仓库目录
        cd wrapper-push
        # 验证复制结果，确认产物完整
        ls -lhR .
        # 提交所有变更，无新修改则跳过（避免工作流报错）
        git add .
        git commit -m "Update from Release ${GITHUB_REF_NAME}: ${GITHUB_SHA:0:7} - ${{ github.event.head_commit.message }}" || echo "No changes to commit"
        # 推送到wrapper-push远程仓库
        git push origin main

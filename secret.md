# 简单解释:如何创建Android签名文件

## 什么是Android签名?

Android应用发布前必须用一个"数字签名"来证明应用的身份,就像给文件盖章一样。 [1](#2-0) 这个签名存储在一个叫`keystore.jks`的文件中。<cite/>

## 您需要做什么

### 第一步:创建签名文件(keystore)

如果您还没有签名文件,需要先创建一个。<cite/>

**在命令行中运行**(Windows用户在PowerShell或命令提示符中):

```bash
keytool -genkey -v -keystore keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias my-key-alias
```

运行后会问您几个问题:
- **密码**(输入两次):这就是`STORE_PASSWORD`和`KEY_PASSWORD`,建议设置相同的密码
- **姓名、组织等信息**:可以随意填写
- **最后确认**:输入`yes`

完成后会在当前目录生成`keystore.jks`文件。<cite/>

### 第二步:转换为base64编码

GitHub Actions需要将这个文件转换成文本格式才能存储。 [2](#2-1) 

**Windows用户**(在PowerShell中):
```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("keystore.jks"))
```

**Mac/Linux用户**:
```bash
base64 -i keystore.jks | tr -d '\n'
```

这会输出一长串字符,复制整个输出内容。<cite/>

### 第三步:在GitHub中添加密钥

1. 打开您的GitHub仓库页面
2. 点击顶部的 **Settings**(设置)
3. 左侧菜单找到 **Secrets and variables** → **Actions**
4. 点击 **New repository secret**(新建仓库密钥)

添加以下4个密钥: [3](#2-2) 

| 密钥名称 | 值 | 说明 |
|---------|---|------|
| `KEYSTORE_BASE64` | 第二步复制的长字符串 | 签名文件的base64编码 |
| `STORE_PASSWORD` | 您设置的密码 | keystore的密码 |
| `KEY_PASSWORD` | 您设置的密码 | key的密码(通常与STORE_PASSWORD相同) |
| `KEY_ALIAS` | `my-key-alias` | 创建时使用的别名 |

### 第四步:重新运行构建

配置完成后,重新推送标签或手动触发GitHub Actions,构建就能成功了。<cite/>

## 如果您要构建TV版本

TV版本需要单独的签名文件和密钥配置: [4](#2-3) 
- 重复上述步骤创建另一个`keystore_tv.jks`
- 添加密钥时使用`TV_`前缀:`TV_KEYSTORE_BASE64`、`TV_STORE_PASSWORD`、`TV_KEY_PASSWORD`、`TV_KEY_ALIAS`

## Notes

- `keytool`是Java自带的工具,如果提示找不到命令,需要先安装Java Development Kit (JDK)
- 签名文件(`keystore.jks`)非常重要,请妥善保管并备份,丢失后无法更新已发布的应用
- 密码也要记住,否则无法使用签名文件
- 这个签名文件只需要创建一次,以后一直使用同一个

Wiki pages you might want to explore:
- [Development and Build System (xiaoyaocz/dart_simple_live)](/wiki/xiaoyaocz/dart_simple_live#6)
- [Platform Configuration (xiaoyaocz/dart_simple_live)](/wiki/xiaoyaocz/dart_simple_live#6.2)

### Citations

**File:** simple_live_app/android/app/build.gradle (L56-64)
```gradle
    signingConfigs {
       release {
           keyAlias keystoreProperties['keyAlias']
           keyPassword keystoreProperties['keyPassword']
           storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
           storePassword keystoreProperties['storePassword']
           v1SigningEnabled true
           v2SigningEnabled true
       }
```

**File:** .github/workflows/publish_app_release.yml (L19-30)
```yaml
      - name: Download Android keystore
        id: android_keystore
        uses: timheuer/base64-to-file@v1.2
        with:
          fileName: keystore.jks
          encodedString: ${{ secrets.KEYSTORE_BASE64 }}
      - name: Create key.properties
        run: |
          echo "storeFile=${{ steps.android_keystore.outputs.filePath }}" > simple_live_app/android/key.properties
          echo "storePassword=${{ secrets.STORE_PASSWORD }}" >> simple_live_app/android/key.properties
          echo "keyPassword=${{ secrets.KEY_PASSWORD }}" >> simple_live_app/android/key.properties
          echo "keyAlias=${{ secrets.KEY_ALIAS }}" >> simple_live_app/android/key.properties
```

**File:** .github/workflows/publish_tv_app_release.yaml (L19-30)
```yaml
      - name: Download Android keystore
        id: android_tv_keystore
        uses: timheuer/base64-to-file@v1.2
        with:
          fileName: keystore.jks
          encodedString: ${{ secrets.TV_KEYSTORE_BASE64 }}
      - name: Create key.properties
        run: |
          echo "storeFile=${{ steps.android_tv_keystore.outputs.filePath }}" > simple_live_tv_app/android/key.properties
          echo "storePassword=${{ secrets.TV_STORE_PASSWORD }}" >> simple_live_tv_app/android/key.properties
          echo "keyPassword=${{ secrets.TV_KEY_PASSWORD }}" >> simple_live_tv_app/android/key.properties
          echo "keyAlias=${{ secrets.TV_KEY_ALIAS }}" >> simple_live_tv_app/android/key.properties
```

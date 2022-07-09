# gradle.properties中定义变量


# gradle.properties中定义变量
```
light_sdk_version = 2.2.5.14
```
## build.gradle中使用
1. 现在gradle中定义一个函数
```
def getProperty(String filename, String propName) {
    def propsFile = rootProject.file(filename)
    if (propsFile.exists()) {
        def props = new Properties()
        props.load(new FileInputStream(propsFile))
        if (props[propName] != null) {
            return props[propName]
        } else {
            print("No such property " + propName + " in file " + filename)
        }
    } else {
        print(filename + " does not exist!")
    }
}
```
2. 使用
```
lightsdk        : "com.tencent.light-withoutso:lightsdk:${getProperty("gradle.properties", "light_sdk_version")}"
```
## 代码中使用
1. 在build.gradle中定义
```
 defaultConfig {
    buildConfigField "String", "LIGHT_SDK_VERSION", "${light_sdk_version}"
 }
```
2. 使用
```
BuildConfig.LIGHT_SDK_VERSION
```

# build.gradle中定义变量
```
ext {
    light_sdk_version = "2.2.5.14"
    light_sdk_component_level = "78"
    pag_sdk_version = "3.3.0.138"
    pag_sdk_tag_level="3"
}
```
## build.gradle中使用
```
 defaultConfig {
    buildConfigField "String", "LIGHT_SDK_VERSION", "\"${project.light_sdk_version}\""
 }
```

## 代码中使用
1. 在build.gradle中定义
```
 defaultConfig {
    buildConfigField "String", "LIGHT_SDK_VERSION", "\"${project.light_sdk_version}\""
 }
```
2. 使用
```
BuildConfig.LIGHT_SDK_VERSION
```

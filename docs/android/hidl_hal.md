# Android hidl hal add new interface 使用详解

> gyniu 发布于 2022-11-11

### 前言

如何在一个已经freeze的 hidl hal 中添加新接口, 比如在struct 中 添加新的字段, 新的方法。


### 基础知识

已经被 freeze的 hidl hal 是不允许添加新的接口的, 解决的办法是继承原有接口, 在新的hidl hal中添加


### 编写 HIDL HAL 接口

编写hidl hal 1.1 版本, 继承原有的 1.0 hidl接口

#### 建立如下结构目录的包名和hidl 文件

```
.
|-- Android.bp
`-- health
    |-- 1.0
    |   |-- Android.bp
    |   |-- IHealth.hal
    |   `-- types.hal
    |-- 1.1
    |   |-- Android.bp
    |   |-- IHealth.hal
    |   `-- types.hal
    |-- Android.bp
    `-- current.txt

3 directories, 9 files

```

#### 1. Android.bp
```java
subdirs = [
    "health",
    "health/1.0",
    "health/1.1",
]

```

#### 2. health/Android.bp
```java
hidl_package_root {
    name: "vendor.gyniu.hardware.health",
    path: "vendor/gyniu/valueadds/interfaces/health",
}

```

#### 3. health/current.txt
```java
ee578b018e0ddbf802828d49a3c8231337c3788b023fbdb4692f5c77de29a3ab vendor.gyniu.hardware.health@1.0::IHealth
fe3a55142d4523d898dea29cb3f892caa3408f24fa48c875405a7d26869a77da vendor.gyniu.hardware.health@1.0::types
35eb82890bc1e6e9ad0ac9c6dfbeef947c9d434cde5495889048b0628f0605cb vendor.gyniu.hardware.health@1.1::types
fc0216e9c08d52d951d9de25644f363d00d3852a86b9c3d8fcb72617156fe5f1 vendor.gyniu.hardware.health@1.1::IHealth

```

#### 4. health/1.1/Android.bp
```java
hidl_interface {
    name: "vendor.gyniu.hardware.health@1.1",
    root: "vendor.gyniu.hardware.health",
    product_specific: true,
    srcs: [
        "types.hal",
        "IHealth.hal",
    ],
    interfaces: [
        "android.hidl.base@1.0",
        "vendor.gyniu.hardware.health@1.0",
    ],
    gen_java: true,
}

```

#### 5. health/1.1/IHealth.hal
```java
package vendor.gyniu.hardware.health@1.1;

import @1.0::IHealth;
import @1.1::GyniuHealthInfo;

interface IHealth extends @1.0::IHealth {

    /*
      Get battery information
      Input Parameter: None
      Output Parameter: An object containing battery information
    */
    getGyniuHealthInfo_1_1() generates (GyniuHealthInfo ret);
};

```

#### 6. health/1.1/types.hal
```java
package vendor.gyniu.hardware.health@1.1;

import @1.0::GyniuHealthInfo;

struct GyniuHealthInfo {
    @1.0::GyniuHealthInfo v1_0;

    int32_t connectorHealth;
};

```

### 实现 HIDL HAL 接口

实现 hidl hal 1.1 版本, 包括原有的 1.0 hidl接口

#### 建立如下结构目录的包名和hidl 文件

```
.
|-- 1.0
|   `-- default
|       |-- Android.bp
|       |-- Health.cpp
|       |-- Health.h
|       |-- service.cpp
|       |-- vendor.gyniu.hardware.health@1.0-service.rc
|       `-- vendor.gyniu.hardware.health@1.0-service.xml
|-- 1.1
|   `-- default
|       |-- Android.bp
|       |-- Health.cpp
|       |-- Health.h
|       |-- service.cpp
|       |-- vendor.gyniu.hardware.health@1.1-service.rc
|       `-- vendor.gyniu.hardware.health@1.1-service.xml
|-- Android.bp
|-- Include.mk
`-- sepolicy
    |-- file.te
    |-- file_contexts
    |-- hal_healthgyniu_hwservice.te
    |-- hwservice.te
    |-- hwservice_contexts
    |-- init.te
    `-- platform_app.te

5 directories, 21 files

```

#### 1. Android.bp
```java
subdirs = [
     "1.0/default",
     "1.1/default",
]

```

#### 2. Include.mk
```java
PRODUCT_PACKAGES += \
        vendor.gyniu.hardware.health@1.1-service \
        vendor.gyniu.hardware.health@1.1-impl

```

#### 3. 1.1/default/Android.bp
```java
cc_binary {
    name: "vendor.gyniu.hardware.health@1.1-service",
    defaults: ["hidl_defaults"],
    vintf_fragments: ["vendor.gyniu.hardware.health@1.1-service.xml"],
    vendor: true,
    relative_install_path: "hw",
    srcs: [
        "Health.cpp",
        "service.cpp",
    ],
    init_rc: ["vendor.gyniu.hardware.health@1.1-service.rc"],
    cflags: [
        "-Wno-unused-parameter",
    ],
    shared_libs: [
        "libhidlbase",
        "libhidltransport",
        "libutils",
        "libhardware",
        "liblog",
        "libcutils",
        "vendor.gyniu.hardware.health@1.0",
        "vendor.gyniu.hardware.health@1.1",
    ],
}

```

#### 4. 1.1/default/Health.cpp
```cpp
#include "Health.h"
#include <cutils/log.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#undef LOG_TAG
#define LOG_TAG "vendor.gyniu.hardware.health@1.1-service"

namespace vendor::gyniu::hardware::health::implementation {

using android::String8;

static int readFromFile(const String8& path, std::string* buf) {
    int fd = 0, ret = -1;
    char val[128] = {'\0'};

    buf->clear();
    fd = open(path, O_RDONLY);
    if (fd < 0) {
        ALOGE("Failed to open %s for reading!\n", path.string());
        return 0;
    }
    ret = read(fd, val, sizeof(val));
    close(fd);
    if (ret < 0) {
        ALOGE("Failed to read %s\n", path.string());
    }
    else {
        buf->append(val, ret);
    }
    return buf->length();
}

static int getIntField(const String8& path) {
    int fd = 0, ret = -1;
    char val[8] = {'\0'};

    fd = open(path, O_RDONLY);
    if (fd < 0) {
        ALOGE("Failed to open %s for reading!\n", path.string());
        return 0;
    }
    ret = read(fd, val, sizeof(val));
    close(fd);
    if (ret < 0) {
        ALOGE("Failed to read %s\n", path.string());
    }
    return (int)atoi(val);

}

void Health::initBatteryProperties()
{
    zHealthInfo.v1_0.batteryMinDischargeTempShutdownLevel = BATTERY_PROPS_INIT_VALUE;
    zHealthInfo.v1_0.batteryMaxDischargeTempShutdownLevel = BATTERY_PROPS_INIT_VALUE;
    zHealthInfo.v1_0.batteryLowLevel = BATTERY_PROPS_INIT_VALUE;
    zHealthInfo.v1_0.batteryCriticalLevel = BATTERY_PROPS_INIT_VALUE;
    zHealthInfo.v1_0.batteryShutdownLevel = BATTERY_PROPS_INIT_VALUE;
    zHealthInfo.v1_0.batteryAdjustShutdownLevel = BATTERY_PROPS_INIT_VALUE;
}

void Health::initBatterySysFsPaths() {
    android::String8 path;
    zHealthConf.batteryMinDischargeTempShutdownPath=String8(String8::kEmptyString);
    if(zHealthConf.batteryMinDischargeTempShutdownPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/temp_min", QCOM_BATT_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryMinDischargeTempShutdownPath = path;
    }
    zHealthConf.batteryMaxDischargeTempShutdownPath=String8(String8::kEmptyString);
    if(zHealthConf.batteryMaxDischargeTempShutdownPath.isEmpty()) {
       path.clear();
       path.appendFormat("%s/temp_max", QCOM_BATT_SYSFS_PATH);
       if (access(path, R_OK) == 0)
           zHealthConf.batteryMaxDischargeTempShutdownPath = path;
    }

    zHealthConf.batteryLowAdjPath = String8(String8::kEmptyString);
    if (zHealthConf.batteryLowAdjPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/battery_low_adj",BATT_EXTRAS_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryLowAdjPath = path;
    }

    zHealthConf.batteryLowPath = String8(String8::kEmptyString);
    if (zHealthConf.batteryLowPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/critical_shutdown", QCOM_BATT_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryLowPath = path;
    }

    zHealthConf.batteryUnfairePath = String8(String8::kEmptyString);
    if (zHealthConf.batteryUnfairePath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/very_low_battery_warning", QCOM_BATT_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryUnfairePath = path;
    }

    zHealthConf.batteryFairePath = String8(String8::kEmptyString);
    if (zHealthConf.batteryFairePath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/low_battery_warning", QCOM_BATT_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryFairePath = path;
    }

    zHealthConf.batteryTypePath = String8(String8::kEmptyString);
    if (zHealthConf.batteryTypePath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/bq27x00-battery/battery_type_int", POWER_SUPPLY_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryTypePath = path;
        else {
            path.clear();
            path.appendFormat("%s/eeprom_battery_profile/battery_type_int", POWER_SUPPLY_SYSFS_PATH);
            if(access(path, R_OK) == 0)
                zHealthConf.batteryTypePath = path;
        }
    }

    zHealthConf.batteryMfdPath = String8(String8::kEmptyString);
    if (zHealthConf.batteryMfdPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/mfd",BATT_EXTRAS_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryMfdPath = path;
    }

    zHealthConf.batteryPartNumberPath = String8(String8::kEmptyString);
    if (zHealthConf.batteryPartNumberPath.isEmpty()) {
       path.clear();
       path.appendFormat("%s/bq27x00-battery/part_number", POWER_SUPPLY_SYSFS_PATH);
       if (access(path, R_OK) == 0)
           zHealthConf.batteryPartNumberPath = path;
       else {
           path.clear();
           path.appendFormat("%s/eeprom_battery_profile/part_number", POWER_SUPPLY_SYSFS_PATH);
           if(access(path, R_OK) == 0)
               zHealthConf.batteryPartNumberPath = path;
       }
    }

    zHealthConf.batterySerialNumberPath = String8(String8::kEmptyString);
    if (zHealthConf.batterySerialNumberPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/serial_number",BATT_EXTRAS_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batterySerialNumberPath = path;
    }

    zHealthConf.batteryRatedCapacityPath = String8(String8::kEmptyString);
    if (zHealthConf.batteryRatedCapacityPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/bq27x00-battery/ratedcapacity", POWER_SUPPLY_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryRatedCapacityPath = path;
        else {
            path.clear();
            path.appendFormat("%s/eeprom_battery_profile/ratedcapacity", POWER_SUPPLY_SYSFS_PATH);
            if (access(path, R_OK) == 0)
                zHealthConf.batteryRatedCapacityPath = path;
        }
    }

    zHealthConf.batteryDecommissionPath = String8(String8::kEmptyString);
    if (zHealthConf.batteryDecommissionPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/bq27x00-battery/battery_health_data1", POWER_SUPPLY_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryDecommissionPath = path;
    }

    zHealthConf.batteryBaseCumulativeChargePath = String8(String8::kEmptyString);
    if (zHealthConf.batteryBaseCumulativeChargePath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/total_accumulated_charge",BATT_EXTRAS_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryBaseCumulativeChargePath = path;
    }

    zHealthConf.batteryUsageNumberPath = String8(String8::kEmptyString);
    if (zHealthConf.batteryUsageNumberPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/cycles",BATT_EXTRAS_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryUsageNumberPath = path;
     }

    zHealthConf.batteryTotalCumulativeChargePath = String8(String8::kEmptyString);
    if (zHealthConf.batteryTotalCumulativeChargePath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/total_cumulative_charge",BATT_EXTRAS_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryTotalCumulativeChargePath = path;
    }

    zHealthConf.batterySecondsSinceFirstUsePath = String8(String8::kEmptyString);
    if (zHealthConf.batterySecondsSinceFirstUsePath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/secs_since_first_use",BATT_EXTRAS_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batterySecondsSinceFirstUsePath = path;
    }

    zHealthConf.batteryPresentCapacityPath = String8(String8::kEmptyString);
    if (zHealthConf.batteryPresentCapacityPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/present_capacity",BATT_EXTRAS_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryPresentCapacityPath = path;
    }

    zHealthConf.batteryHealthPercentagePath = String8(String8::kEmptyString);
    if (zHealthConf.batteryHealthPercentagePath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/bq27x00-battery/soh", POWER_SUPPLY_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryHealthPercentagePath = path;
    }

    zHealthConf.batteryTimetoEmptyPath= String8(String8::kEmptyString);
    if (zHealthConf.batteryTimetoEmptyPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/time_to_suspend",BATT_EXTRAS_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryTimetoEmptyPath = path;
     }

     zHealthConf.batteryTimetoFullPath = String8(String8::kEmptyString);
     if (zHealthConf.batteryTimetoFullPath.isEmpty()) {
         path.clear();
         path.appendFormat("%s/time_to_full",BATT_EXTRAS_SYSFS_PATH);
         if (access(path, R_OK) == 0)
             zHealthConf.batteryTimetoFullPath = path;
     }

    zHealthConf.batteryPresentChargePath = String8(String8::kEmptyString);
    if (zHealthConf.batteryPresentChargePath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/bq27x00-battery/energy_now", POWER_SUPPLY_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryPresentChargePath = path;
    }

    zHealthConf.batteryPercentDecommissionThresholdPath = String8(String8::kEmptyString);
    if (zHealthConf.batteryPercentDecommissionThresholdPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/percent_decommission_threshold",BATT_EXTRAS_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryPercentDecommissionThresholdPath = path;
    }

    zHealthConf.batteryUsageDecommissionThresholdPath = String8(String8::kEmptyString);
    if (zHealthConf.batteryUsageDecommissionThresholdPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/batteryusage_decommission_threshold",BATT_EXTRAS_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryUsageDecommissionThresholdPath = path;
    }

    zHealthConf.batteryErrorStausPath = String8(String8::kEmptyString);
    if (zHealthConf.batteryErrorStausPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/battery/battery_errors",POWER_SUPPLY_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryErrorStausPath = path;
    }

    zHealthConf.batterySocExternalPath = String8(String8::kEmptyString);
    if (zHealthConf.batterySocExternalPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/soc",BATT_EXTRAS_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batterySocExternalPath = path;
    }

    zHealthConf.batterySocInternalPath = String8(String8::kEmptyString);
    if (zHealthConf.batterySocInternalPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/capacity",QCOM_BATT_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batterySocInternalPath = path;
    }

    zHealthConf.batteryTempExternalPath = String8(String8::kEmptyString);
    if (zHealthConf.batteryTempExternalPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/temp",BATT_EXTRAS_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryTempExternalPath = path;
    }

    zHealthConf.batteryTempInternalPath = String8(String8::kEmptyString);
    if (zHealthConf.batteryTempInternalPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/temp",QCOM_BATT_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.batteryTempInternalPath = path;
    }

/*Add by T2M for GROOTT-723 [Begin]*/
    zHealthConf.connectorHealthPath = String8(String8::kEmptyString);
    if (zHealthConf.connectorHealthPath.isEmpty()) {
        path.clear();
        path.appendFormat("%s/usb/connector_health", POWER_SUPPLY_SYSFS_PATH);
        if (access(path, R_OK) == 0)
            zHealthConf.connectorHealthPath = path;
    }
/*Add by T2M for GROOTT-723 [End]*/

}

void Health::getBattExtras()
{
    std::string buf;

    if (!zHealthConf.batteryMinDischargeTempShutdownPath.isEmpty())
        zHealthInfo.v1_0.batteryMinDischargeTempShutdownLevel=getIntField((zHealthConf.batteryMinDischargeTempShutdownPath));

    if (!zHealthConf.batteryMaxDischargeTempShutdownPath.isEmpty())
        zHealthInfo.v1_0.batteryMaxDischargeTempShutdownLevel=getIntField((zHealthConf.batteryMaxDischargeTempShutdownPath));

    if (!zHealthConf.batteryFairePath.isEmpty())
        zHealthInfo.v1_0.batteryLowLevel = getIntField(zHealthConf.batteryFairePath);

    if (!zHealthConf.batteryUnfairePath.isEmpty())
        zHealthInfo.v1_0.batteryCriticalLevel = getIntField(zHealthConf.batteryUnfairePath);

    if (!zHealthConf.batteryLowPath.isEmpty())
        zHealthInfo.v1_0.batteryShutdownLevel = getIntField(zHealthConf.batteryLowPath);

    if (!zHealthConf.batteryLowAdjPath.isEmpty())
        zHealthInfo.v1_0.batteryAdjustShutdownLevel = getIntField(zHealthConf.batteryLowAdjPath);

    if (!zHealthConf.batteryTypePath.isEmpty())
        zHealthInfo.v1_0.batteryType = getIntField(zHealthConf.batteryTypePath);
    else
        zHealthInfo.v1_0.batteryType = 204;

    if (!zHealthConf.batteryMfdPath.isEmpty() && (readFromFile(zHealthConf.batteryMfdPath, &buf)>0))
        zHealthInfo.v1_0.batteryMfd=String8(buf.c_str());

    if (!zHealthConf.batteryPartNumberPath.isEmpty() && (readFromFile(zHealthConf.batteryPartNumberPath, &buf)>0))
        zHealthInfo.v1_0.batteryPartNumber = String8(buf.c_str());

    if (!zHealthConf.batterySerialNumberPath.isEmpty() && (readFromFile(zHealthConf.batterySerialNumberPath, &buf)>0))
        zHealthInfo.v1_0.batterySerialNumber = String8(buf.c_str());

    if (!zHealthConf.batteryRatedCapacityPath.isEmpty())
        zHealthInfo.v1_0.batteryRatedCapacity = getIntField(zHealthConf.batteryRatedCapacityPath);

    if (!zHealthConf.batteryDecommissionPath.isEmpty()) {
        zHealthInfo.v1_0.batteryDecommission = getIntField(zHealthConf.batteryDecommissionPath);
        /*
        if(BATT_DECOMMISSION_HEALTHY_SYSFS == zHealthInfo.v1_0.batteryDecommission) {
           zHealthInfo.v1_0.batteryDecommission = BATT_DECOMMISSION_HEALTHY;
         } else if (BATT_DECOMMISSION_UNHEALTHY_SYFS == zHealthInfo.v1_0.batteryDecommission) {
              zHealthInfo.v1_0.batteryDecommission = BATT_DECOMMISSION_UNHEALTHY;
         } else {
              zHealthInfo.v1_0.batteryDecommission = BATT_DECOMMISSION_UNKNOWN;
         }
         */
         // Health percentage is zero on FG reset. Added a check to show health percentage as 255% and decomission status as unknown.
         if(0 == zHealthInfo.v1_0.batteryHealthPercentage) {
             zHealthInfo.v1_0.batteryDecommission = BATT_DECOMMISSION_UNKNOWN;
             zHealthInfo.v1_0.batteryHealthPercentage = 255;
         }
    } else
        zHealthInfo.v1_0.batteryDecommission = BATT_DECOMMISSION_UNKNOWN;

    if (!zHealthConf.batteryUsageNumberPath.isEmpty())
            zHealthInfo.v1_0.batteryUsageNumber = getIntField(zHealthConf.batteryUsageNumberPath);

    if (!zHealthConf.batteryBaseCumulativeChargePath.isEmpty())
        zHealthInfo.v1_0.batteryBaseCumulativeCharge = getIntField(zHealthConf.batteryBaseCumulativeChargePath);
    else
        zHealthInfo.v1_0.batteryBaseCumulativeCharge = zHealthInfo.v1_0.batteryRatedCapacity * zHealthInfo.v1_0.batteryUsageNumber;

    if (!zHealthConf.batteryTotalCumulativeChargePath.isEmpty())
        zHealthInfo.v1_0.batteryTotalCumulativeCharge = getIntField(zHealthConf.batteryTotalCumulativeChargePath);
    else
        zHealthInfo.v1_0.batteryBaseCumulativeCharge = zHealthInfo.v1_0.batteryRatedCapacity * zHealthInfo.v1_0.batteryUsageNumber;

    // the extra fields expected, but not Time Since First Use. The UI will
    // hide data that is marked as invalid, so return EINVAL rather than -1.
    if (!zHealthConf.batterySecondsSinceFirstUsePath.isEmpty())
        zHealthInfo.v1_0.batterySecondsSinceFirstUse = getIntField(zHealthConf.batterySecondsSinceFirstUsePath);
    else
        zHealthInfo.v1_0.batterySecondsSinceFirstUse = -EINVAL;

    if (!zHealthConf.batteryPresentCapacityPath.isEmpty())
        zHealthInfo.v1_0.batteryPresentCapacity = getIntField(zHealthConf.batteryPresentCapacityPath);

    if (!zHealthConf.batteryHealthPercentagePath.isEmpty()) {
        zHealthInfo.v1_0.batteryHealthPercentage = getIntField(zHealthConf.batteryHealthPercentagePath);
        zHealthInfo.v1_0.batteryPresentCapacity = zHealthInfo.v1_0.batteryRatedCapacity * zHealthInfo.v1_0.batteryHealthPercentage;
    }

    if (!zHealthConf.batteryTimetoEmptyPath.isEmpty())
        zHealthInfo.v1_0.batteryTimetoEmpty = getIntField(zHealthConf.batteryTimetoEmptyPath);

    if (!zHealthConf.batteryTimetoFullPath.isEmpty())
        zHealthInfo.v1_0.batteryTimetoFull = getIntField(zHealthConf.batteryTimetoFullPath) / 60;
    else
        zHealthInfo.v1_0.batteryTimetoFull = 0;

    if (!zHealthConf.batteryPresentChargePath.isEmpty())
        zHealthInfo.v1_0.batteryPresentCharge = getIntField(zHealthConf.batteryPresentChargePath);

    if (!zHealthConf.batteryPercentDecommissionThresholdPath.isEmpty())
        zHealthInfo.v1_0.batteryPercentDecommissionThreshold = getIntField(zHealthConf.batteryPercentDecommissionThresholdPath);

    if (!zHealthConf.batteryUsageDecommissionThresholdPath.isEmpty())
        zHealthInfo.v1_0.batteryUsageDecommissionThreshold = getIntField(zHealthConf.batteryUsageDecommissionThresholdPath);

    if (!zHealthConf.batteryErrorStausPath.isEmpty())
        zHealthInfo.v1_0.batteryErrorStatus = getIntField(zHealthConf.batteryErrorStausPath);

    if (!zHealthConf.batterySocExternalPath.isEmpty())
        zHealthInfo.v1_0.batterySocExternal = getIntField(zHealthConf.batterySocExternalPath);

    if (!zHealthConf.batterySocInternalPath.isEmpty())
        zHealthInfo.v1_0.batterySocInternal = getIntField(zHealthConf.batterySocInternalPath);

    if (!zHealthConf.batteryTempExternalPath.isEmpty())
        zHealthInfo.v1_0.batteryTempExternal = getIntField(zHealthConf.batteryTempExternalPath);

    if (!zHealthConf.batteryTempInternalPath.isEmpty())
        zHealthInfo.v1_0.batteryTempInternal = getIntField(zHealthConf.batteryTempInternalPath);
}

Health::Health(){
    initBatterySysFsPaths();
    initBatteryProperties();
    ALOGI("v1_1 HealthGyniu initialized.");
}

// Methods from ::vendor::gyniu::hardware::health::V1_0::IHealth follow.
Return<void> Health::getGyniuHealthInfo(getGyniuHealthInfo_cb _hidl_cb) {
    ALOGI("v1_1 getGyniuHealthInfo");
    getBattExtras();
    _hidl_cb(zHealthInfo.v1_0);
    return Void();
}

Return<void> Health::getGyniuHealthInfo_1_1(getGyniuHealthInfo_1_1_cb _hidl_cb) {
    ALOGI("v1_1 getGyniuHealthInfo_1_1");
    getBattExtras();

    /*Add by T2M for GROOTT-723 [Begin]*/
    if (!zHealthConf.connectorHealthPath.isEmpty())
        zHealthInfo.connectorHealth = getIntField(zHealthConf.connectorHealthPath);
    /*Add by T2M [end]*/

    _hidl_cb(zHealthInfo);
    return Void();
}

Return<int32_t> Health::getBatteryCriticalLevel() {
    if (!zHealthConf.batteryUnfairePath.isEmpty())
        return getIntField(zHealthConf.batteryUnfairePath);
    else
        return BATTERY_PROPS_INIT_VALUE;
}

Return<int32_t> Health::getBatteryLowLevel() {
    if (!zHealthConf.batteryFairePath.isEmpty())
        return getIntField(zHealthConf.batteryFairePath);
    else
        return BATTERY_PROPS_INIT_VALUE;
}


// Methods from ::android::hidl::base::V1_0::IBase follow.

//IHealth* HIDL_FETCH_IHealth(const char* /* name */) {
    //return new Health();
//}
//
}  // namespace vendor::gyniu::hardware::health::implementation

```

#### 5. 1.1/default/Health.h
```cpp
#pragma once

#include <vendor/gyniu/hardware/health/1.1/IHealth.h>
#include <hidl/MQDescriptor.h>
#include <hidl/Status.h>
#include <utils/String8.h>
#include <stdbool.h>

#define POWER_SUPPLY_SYSFS_PATH "/sys/class/power_supply"
#define BATT_EXTRAS_SYSFS_PATH "/sys/class/power_supply/batt_extras"
#define QCOM_BATT_SYSFS_PATH "/sys/class/power_supply/battery"
#define BATT_DECOMMISSION_HEALTHY_SYSFS        0x64
#define BATT_DECOMMISSION_UNHEALTHY_SYFS       0xFE
#define BATT_DECOMMISSION_UNKNOWN              0x2
#define BATT_DECOMMISSION_HEALTHY              0x0
#define BATT_DECOMMISSION_UNHEALTHY            0x1
#define BATTERY_PROPS_INIT_VALUE               255

namespace vendor::gyniu::hardware::health::implementation {

using ::android::hardware::hidl_array;
using ::android::hardware::hidl_memory;
using ::android::hardware::hidl_string;
using ::android::hardware::hidl_vec;
using ::android::hardware::Return;
using ::android::hardware::Void;
using ::android::sp;
using android::String8;

struct GyniuHealthdConfig {
    android::String8 batteryMinDischargeTempShutdownPath;
    android::String8 batteryMaxDischargeTempShutdownPath;
    android::String8 batteryFairePath;
    android::String8 batteryUnfairePath;
    android::String8 batteryLowPath;
    android::String8 batteryLowAdjPath;
    android::String8 batteryTypePath;
    android::String8 batteryMfdPath;
    android::String8 batteryPartNumberPath;
    android::String8 batterySerialNumberPath;
    android::String8 batteryRatedCapacityPath;
    android::String8 batteryDecommissionPath;
    android::String8 batteryBaseCumulativeChargePath;
    android::String8 batteryUsageNumberPath;
    android::String8 batteryTotalCumulativeChargePath;
    android::String8 batterySecondsSinceFirstUsePath;
    android::String8 batteryPresentCapacityPath;
    android::String8 batteryHealthPercentagePath;
    android::String8 batteryTimetoEmptyPath;
    android::String8 batteryTimetoFullPath;
    android::String8 batteryPresentChargePath;
    android::String8 batteryPercentDecommissionThresholdPath;
    android::String8 batteryUsageDecommissionThresholdPath;
    android::String8 batteryErrorStausPath;
    android::String8 batterySocExternalPath;
    android::String8 batterySocInternalPath;
    android::String8 batteryTempExternalPath;
    android::String8 batteryTempInternalPath;
    android::String8 connectorHealthPath;     //Add by T2M for GROOTT-723
};

struct Health : public V1_1::IHealth {
    Health();
    Return<void> getGyniuHealthInfo(getGyniuHealthInfo_cb _hidl_cb) override;
    Return<void> getGyniuHealthInfo_1_1(getGyniuHealthInfo_1_1_cb _hidl_cb) override;
    Return<int32_t> getBatteryCriticalLevel() override;
    Return<int32_t> getBatteryLowLevel() override;

    // Methods from ::android::hidl::base::V1_0::IBase follow.
private:
    void initBatteryProperties();
    void initBatterySysFsPaths();
    void getBattExtras();
    struct V1_1::GyniuHealthInfo zHealthInfo;
    struct GyniuHealthdConfig zHealthConf;
};

// FIXME: most likely delete, this is only for passthrough implementations
// extern "C" IHealth* HIDL_FETCH_IHealth(const char* name);

}  // namespace vendor::gyniu::hardware::health::implementation

```

#### 6. 1.1/default/service.cpp
```cpp
#include <hidl/HidlSupport.h>
#include <hidl/HidlTransportSupport.h>
#include <hidl/LegacySupport.h>
#include "Health.h"

using ::android::hardware::configureRpcThreadpool;
using ::vendor::gyniu::hardware::health::V1_1::IHealth;
using ::vendor::gyniu::hardware::health::implementation::Health;
using ::android::hardware::joinRpcThreadpool;
using ::android::OK;
using ::android::sp;

int main(int /* argc */, char* /* argv */ []) {
    int res;
    android::sp<IHealth> ser = new Health();
    ALOGI("V1_1::IHealth main");
    configureRpcThreadpool(1, true /*callerWillJoin*/);

    if (ser != nullptr) {
        res = ser->registerAsService();
        if(res != 0)
              ALOGE("Cant register instance of V1_1::IHealth, nullptr");
    } else {
         ALOGE("Can't create instance of V1_1::IHealth, nullptr");
    }

    joinRpcThreadpool();

    return 0; // should never get here
}

```

#### 7. 1.1/default/vendor.gyniu.hardware.health@1.1-service.rc
```java

service healthgyniu-hal-1-1 /vendor/bin/hw/vendor.gyniu.hardware.health@1.1-service
     interface vendor.gyniu.hardware.health@1.0::IHealth default
     interface vendor.gyniu.hardware.health@1.1::IHealth default
     class hal
     user system
     group system

```

#### 8. 1.1/default/vendor.gyniu.hardware.health@1.1-service.xml
```xml
<manifest version="1.0" type="device">
    <hal format="hidl">
        <name>vendor.gyniu.hardware.health</name>
        <transport>hwbinder</transport>
        <version>1.1</version>
        <interface>
            <name>IHealth</name>
            <instance>default</instance>
        </interface>
    </hal>
</manifest>

```

#### 9. sepolicy/file_contexts

    /(vendor|system/vendor)/bin/hw/vendor\.gyniu\.hardware\.health@1\.1-service           u:object_r:hal_healthgyniu_hwservice_exec:s0

### 使用方式修改

```java
diff --git a/packages/SystemUI/Android.bp b/packages/SystemUI/Android.bp
index 036e5c6aa02b..19dfa0a4a5be 100644
--- a/packages/SystemUI/Android.bp
+++ b/packages/SystemUI/Android.bp
@@ -124,6 +124,7 @@ android_library {
         "jsr330",
         "lottie",
         "vendor.gyniu.hardware.health-V1.0-java",
+        "vendor.gyniu.hardware.health-V1.1-java",
     ],
     manifest: "AndroidManifest.xml",
     defaults: [
diff --git a/services/core/Android.bp b/services/core/Android.bp
index f531d7e0c706..3653ba954891 100644
--- a/services/core/Android.bp
+++ b/services/core/Android.bp
@@ -158,6 +158,7 @@ java_library_static {
         "android.hardware.health-V1-java", // AIDL
         "android.hardware.health-translate-java",
         "vendor.gyniu.hardware.health-V1.0-java",
+        "vendor.gyniu.hardware.health-V1.1-java",
         "android.hardware.light-V1-java",
         "android.hardware.tv.cec-V1.1-java",
         "android.hardware.weaver-V1.0-java",
diff --git a/services/core/java/com/android/server/BatteryService.java b/services/core/java/com/android/server/BatteryService.java
index d44f151412e1..5e01cac67375 100755
--- a/services/core/java/com/android/server/BatteryService.java
+++ b/services/core/java/com/android/server/BatteryService.java
@@ -61,6 +61,7 @@ import android.service.battery.BatteryServiceDumpProto;
 import android.sysprop.PowerProperties;
 import android.util.EventLog;
 import android.util.Slog;
+import android.util.Singleton;
 import android.util.proto.ProtoOutputStream;

 import com.android.internal.app.IBatteryStats;
@@ -177,6 +178,8 @@ public final class BatteryService extends SystemService {
     private final HealthInfo mLastHealthInfo = new HealthInfo();
     private vendor.gyniu.hardware.health.V1_0.GyniuHealthInfo mGyniuHealthInfo;
     private vendor.gyniu.hardware.health.V1_0.IHealth mHealthGyniu;
+    private vendor.gyniu.hardware.health.V1_1.GyniuHealthInfo mGyniuHealthInfoV11;
+    private vendor.gyniu.hardware.health.V1_1.IHealth mHealthGyniuV11;
     private boolean mBatteryLevelCritical;
     private int mChargingLedNotification = CHARGING_LED_DISABLE;
     private int mLastBatteryStatus;
@@ -291,8 +294,16 @@ public final class BatteryService extends SystemService {
                     com.android.internal.R.integer.config_defaultshutdownBatteryminTemp);

             try {
-                mHealthGyniu = vendor.gyniu.hardware.health.V1_0.IHealth.getService(true);
-                mGyniuHealthInfo = mHealthGyniu.getGyniuHealthInfo();
+                mHealthGyniu = getService();
+                mHealthGyniuV11 =
+                    vendor.gyniu.hardware.health.V1_1.IHealth.castFrom(mHealthGyniu);
+                if (mHealthGyniuV11 != null) {
+                    mGyniuHealthInfoV11 = mHealthGyniuV11.getGyniuHealthInfo_1_1();
+                    mGyniuHealthInfo = mGyniuHealthInfoV11.v1_0;
+                } else {
+                    mGyniuHealthInfo = mHealthGyniu.getGyniuHealthInfo();
+                }
+
                 if(DEBUG) {
                     Slog.d(TAG, "GyniuHealthInfor Battery type:"+mGyniuHealthInfo.batteryType
                                 +" Capacity:"+mGyniuHealthInfo.batteryPresentCapacity);
@@ -300,6 +311,9 @@ public final class BatteryService extends SystemService {
                                 +" Serial#"+mGyniuHealthInfo.batterySerialNumber);
                     Slog.d(TAG, "GyniuHealthInfor Low level:"+mHealthGyniu.getBatteryLowLevel()
                                 +" Critical level:"+mHealthGyniu.getBatteryCriticalLevel());
+                    if (mHealthGyniuV11 != null) {
+                        Slog.d(TAG, "GyniuHealthInfor connect health#:" + mGyniuHealthInfoV11.connectorHealth);
+                    }
                 }
             } catch (RemoteException e) {
                 e.printStackTrace();
@@ -307,6 +321,36 @@ public final class BatteryService extends SystemService {
         }
     }

+    private static final Singleton<vendor.gyniu.hardware.health.V1_0.IHealth> sService =
+            new Singleton<vendor.gyniu.hardware.health.V1_0.IHealth>() {
+        @Override
+        protected vendor.gyniu.hardware.health.V1_0.IHealth create() {
+            try {
+                Slog.d(TAG, "Trying to get IHealth@1.1 service");
+                vendor.gyniu.hardware.health.V1_1.IHealth serviceV11 =
+                        vendor.gyniu.hardware.health.V1_1.IHealth.getService(true /*wait*/);
+                if (serviceV11 != null) {
+                    return serviceV11;
+                }
+            } catch (Exception eV1_1) {
+                Slog.d(TAG, "Failed to get IHealth@1.1 service");
+            }
+
+            try {
+                Slog.d(TAG, "Trying to get IHealth@1.0 service");
+                return vendor.gyniu.hardware.health.V1_0.IHealth.getService(true /*wait*/);
+            } catch (Exception eV1_0) {
+                Slog.d(TAG, "Failed to get IHealth@1.0 service");
+            }
+
+            return null;
+        }
+    };
+
+    static vendor.gyniu.hardware.health.V1_0.IHealth getService() {
+        return sService.get();
+    }
+
     @Override
     public void onStart() {
         registerHealthCallback();
@@ -630,7 +674,13 @@ public final class BatteryService extends SystemService {
                 // Process the new values.
                 if(GyniuUtils.isGyniu().orElse(false)) {
                     try {
-                        mGyniuHealthInfo = mHealthGyniu.getGyniuHealthInfo();
+                        if (mHealthGyniuV11 != null) {
+                            mGyniuHealthInfoV11 = mHealthGyniuV11.getGyniuHealthInfo_1_1();
+                            mGyniuHealthInfo = mGyniuHealthInfoV11.v1_0;
+                        } else {
+                            Slog.d(TAG, "mHealthGyniuV11 is null");
+                            mGyniuHealthInfo = mHealthGyniu.getGyniuHealthInfo();
+                        }
                         processValuesLocked(false);
                     } catch (RemoteException e) {
                         e.printStackTrace();

```

### 版本兼容

#### 1. vendor/qcom/opensource/core-utils/vendor_framework_compatibility_matrix.xml

```xml
    <hal format="hidl" optional="true">
        <name>vendor.gyniu.hardware.health</name>
        <version>1.0-1</version>
        <interface>
            <name>IHealth</name>
            <instance>default</instance>
        </interface>
    </hal>

```

### QIIFA编译报错, two solution

#### 1. vendor/qcom/proprietary/commonsys-intf/QIIFA-cmd/hidl/qiifa_hidl_skipped_interfaces.txt

```java
vendor.gyniu.hardware.health

```

#### 2. or add your hidl hal info to json file 
    
    vendor/qcom/proprietary/commonsys-intf/QIIFA-cmd/hidl/
        qiifa_hidl_cmd.json*
        qiifa_hidl_cmd_3.json*
        qiifa_hidl_cmd_4.json*
        qiifa_hidl_cmd_5.json*
        qiifa_hidl_cmd_6.json*
        qiifa_hidl_cmd_7.json*
        qiifa_hidl_cmd_device.json*


### FAQ

Q:添加了HIDL hal,如何生成对应的 hash 值，copy到current.txt中

A: 执行下面的操作：

    找到 hidl_package_root 的定义

    hidl-gen -L hash -r$NAME:$PATH -randroid.hidl:system/libhidl/transport $PACKAGE

    比如:
    hidl-gen -L hash -rvendor.gyniu.hardware.health:vendor/gyniu/valueadds/interfaces/health -randroid.hidl:system/libhidl/transport vendor.gyniu.hardware.health@1.1


> 可以将上面的命令放在一个shell脚本中,方便复用



Q:下面的报错如何处理1

    E hwservicemanager: getDeviceHalManifest: -2147483648 VINTF parse error: Cannot add manifest fragment /vendor/etc/vintf/manifest/vendor.gyniu.hardware.health@1.0-service.xml:HAL "vendor.gyniu.hardware.health" has a conflict.


A: 解决办法：

    如果 vendor.gyniu.hardware.health 声明在device的manifest 中，将低版本移除即可
    如果 vendor.gyniu.hardware.health 声明在一个单独文件, 不要编译copy低版本 xml 到 vendor/etc/vintf/ 下




Q:下面的报错如何处理2

    host_init_verifier: device/t2m/common/battery/health/1.1/default/vendor.gyniu.hardware.health@1.1-service.rc: 15: 
    Interface 'vendor.gyniu.hardware.health@1.1::IHealth' requires its full inheritance hierarchy to be listed in this init_rc file. Missing interfaces: [vendor.gyniu.hardware.health@1.0::IHealth]
    host_init_verifier: Failed to parse init script 'device/t2m/common/battery/health/1.1/default/vendor.gyniu.hardware.health@1.1-service.rc' with 1 errors

A: 解决办法：

    添加: interface vendor.gyniu.hardware.health@1.0::IHealth default
    到高版本的rc文件 vendor.gyniu.hardware.health@1.1-service.rc

    service healthgyniu-hal-1-1 /vendor/bin/hw/vendor.gyniu.hardware.health@1.1-service
    interface vendor.gyniu.hardware.health@1.0::IHealth default
    interface vendor.gyniu.hardware.health@1.1::IHealth default
    class hal
    user system
    group system


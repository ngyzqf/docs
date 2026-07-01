# Android aidl hal 使用详解

> gyniu 发布于 2022-08-30

### 前言

Android 11 引入了在 Android 中使用 AIDL 实现 HAL 的功能。这样就可以在不使用 HIDL 的情况下实现 Android 的部分代码。

如果 HAL 使用 AIDL 在框架组件（例如 system.img 中的组件）和硬件组件（例如 vendor.img 中的组件）之间进行通信，则必须使用稳定的 AIDL。不过，如需在分区内进行通信（例如从一个 HAL 到另一个），则对所使用的 IPC 机制没有任何限制。

AIDL 出现在 HIDL 之前，而且在 Android 框架组件之间或应用内等其他很多地方都有使用。现在，由于 AIDL 具备了稳定性支持，就可以只使用一个 IPC 运行时环境实现整个栈。此外，AIDL 的版本控制系统也优于 HIDL。

### 基础知识

为了让系统和供应商都可使用某个 AIDL 接口，需要对该接口进行两项更改：

* 每个类型定义都需要带有 `@VintfStability` 注释
* `aidl_interface` 声明需要包含 `stability: "vintf",`

只有接口所有者可以进行这些更改。

此外，为了最大限度地提高代码可移植性并避免出现潜在问题(例如不必要的额外库),建议停用 CPP 后端.
```
backend: {
    cpp: {
        enabled: false,
    },
},

```

### 编写 AIDL HAL 接口

#### 建立如下结构目录的包名和aidl 文件

```
.
`-- GynService
    |-- Android.bp
    |-- aidl_api
    |   `-- com.ftd.gyn
    |       |-- 1
    |       |   `-- com
    |       |       `-- ftd
    |       |           `-- gyn
    |       |               `-- IGyndService.aidl
    |       `-- current
    |           `-- com
    |               `-- ftd
    |                   `-- gyn
    |                       `-- IGyndService.aidl
    |-- com
    |   `-- ftd
    |       `-- gyn
    |           `-- IGyndService.aidl
    `-- default
        |-- Android.bp
        |-- com.ftd.gyn.rc
        |-- com.ftd.gyn.xml
        |-- daemon.cpp
        |-- gyn_service.cpp
        `-- include
            `-- gyn_service.h

16 directories, 10 files

```

#### 1. Android.bp
```java
aidl_interface {
    name: "com.ftd.gyn",
    vendor_available: true,
    srcs: [":gyn_aidl"],
    stability: "vintf",
    backend: {
        java: {
            platform_apis: true,
        },
        cpp: {
            enabled: false,
        },
    },
    versions: ["1"],
}

/// for common default impl
filegroup {
    name: "gyn_aidl",
    srcs: [
        "com/ftd/gyn/IGyndService.aidl",
    ],
    path: "com",
}

```

#### 2. aidl_api/com.ftd.gyn/1/com/ftd/gyn/IGyndService.aidl
```java
///////////////////////////////////////////////////////////////////////////////
// THIS FILE IS IMMUTABLE. DO NOT EDIT IN ANY CASE.                          //
///////////////////////////////////////////////////////////////////////////////

// This file is a snapshot of an AIDL interface (or parcelable). Do not try to
// edit this file. It looks like you are doing that because you have modified
// an AIDL interface in a backward-incompatible way, e.g., deleting a function
// from an interface or a field from a parcelable and it broke the build. That
// breakage is intended.
//
// You must not make a backward incompatible changes to the AIDL files built
// with the aidl_interface module type with versions property set. The module
// type is used to build AIDL files in a way that they can be used across
// independently updatable components of the system. If a device is shipped
// with such a backward incompatible change, it has a high risk of breaking
// later when a module using the interface is updated, e.g., Mainline modules.

package com.ftd.gyn;
@VintfStability
interface IGyndService {
  int open(in @utf8InCpp String path);
  oneway void close(in int fd);
}

```

#### 3. aidl_api/com.ftd.gyn/current/com/ftd/gyn/IGyndService.aidl
```java
///////////////////////////////////////////////////////////////////////////////
// THIS FILE IS IMMUTABLE. DO NOT EDIT IN ANY CASE.                          //
///////////////////////////////////////////////////////////////////////////////

// This file is a snapshot of an AIDL interface (or parcelable). Do not try to
// edit this file. It looks like you are doing that because you have modified
// an AIDL interface in a backward-incompatible way, e.g., deleting a function
// from an interface or a field from a parcelable and it broke the build. That
// breakage is intended.
//
// You must not make a backward incompatible changes to the AIDL files built
// with the aidl_interface module type with versions property set. The module
// type is used to build AIDL files in a way that they can be used across
// independently updatable components of the system. If a device is shipped
// with such a backward incompatible change, it has a high risk of breaking
// later when a module using the interface is updated, e.g., Mainline modules.

package com.ftd.gyn;
@VintfStability
interface IGyndService {
  int open(in @utf8InCpp String path);
  oneway void close(in int fd);
}

```

#### 4. com/ftd/gyn/IGyndService.aidl
```java
/*
 * Copyright (C) 2022 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package com.ftd.gyn;

@VintfStability
interface IGyndService {
    int open(in @utf8InCpp String path);
    oneway void close(in int fd);
}

```

#### 5. default/Android.bp
```java
cc_library_shared {
    name: "com.ftd.gyn.impl",
    vendor: true,
    srcs: [
        "gyn_service.cpp",
    ],
    shared_libs: [
        "liblog",
        "libutils",
        "libcutils",
        "libbase",
        "libui",
        "libbinder",
        "libhidlbase",
        "libbinder_ndk",
        "com.ftd.gyn-ndk_platform",
    ],
    cflags: [
        "-Wall",
        "-Wextra",
        "-Werror",
        "-Wno-unused-parameter",
    ],
    export_include_dirs: ["include"]
}

cc_binary {
    name: "gynd",
    vendor: true,
    srcs: [
        "daemon.cpp",
    ],
    shared_libs: [
        "liblog",
        "libutils",
        "libcutils",
        "libbase",
        "libui",
        "libbinder",
        "libbinder_ndk",
        "libhidlbase",
        "com.ftd.gyn-ndk_platform",
        "com.ftd.gyn.impl",
    ],
    cflags: [
        "-Wall",
        "-Wextra",
        "-Werror",
        "-Wno-unused-parameter",
    ],
    init_rc: ["com.ftd.gyn.rc"],
    vintf_fragments: [
        "com.ftd.gyn.xml",
    ],
    local_include_dirs: ["include"],
}

```

#### 6. default/com.ftd.gyn.rc
```java
service hal_gynd_default /vendor/bin/gynd
    class hal
    user system
    group system

```

#### 7. default/com.ftd.gyn.xml
```xml
<!-- Copyright (c) 2022 The Linux Foundation. All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are
met:
    * Redistributions of source code must retain the above copyright
      notice, this list of conditions and the following disclaimer.
    * Redistributions in binary form must reproduce the above
      copyright notice, this list of conditions and the following
      disclaimer in the documentation and/or other materials provided
      with the distribution.
    * Neither the name of The Linux Foundation nor the names of its
      contributors may be used to endorse or promote products derived
      from this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED "AS IS" AND ANY EXPRESS OR IMPLIED
WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NON-INFRINGEMENT
ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS
BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR
BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE
OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN
IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
-->
<manifest version="1.0" type="device">
    <hal format="aidl">
        <name>com.ftd.gyn</name>
        <fqname>IGyndService/default</fqname>
    </hal>
</manifest>

```

#### 8. default/daemon.cpp
```cpp
//
// Copyright (C) 2022 The Android Open Source Project
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//      http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
//

#include <android-base/logging.h>
#include <android/binder_manager.h>
#include <android/binder_process.h>

#include "gyn_service.h"

using aidl::com::ftd::gyn::GyndService;

int main(int argc, char** argv) {
    ABinderProcess_setThreadPoolMaxThreadCount(0);
    std::shared_ptr<GyndService> service = ndk::SharedRefBase::make<GyndService>();

    const std::string instance = std::string() + GyndService::descriptor + "/default";

    LOG(ERROR) << "Register Service: " << instance.c_str();

    binder_status_t status = AServiceManager_addService(service->asBinder().get(), instance.c_str());
    CHECK(status == STATUS_OK);

    ABinderProcess_joinThreadPool();
    return EXIT_FAILURE;  // should not reach
}

```

#### 9. default/gyn_service.cpp
```cpp
/*
 * Copyright (C) 2019 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#include "gyn_service.h"

#include <errno.h>
#include <linux/fs.h>
#include <stdio.h>
#include <sys/ioctl.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <sys/vfs.h>
#include <unistd.h>

#include <array>
#include <chrono>
#include <string>
#include <vector>

#include <android-base/errors.h>
#include <android-base/file.h>
#include <android-base/logging.h>
#include <android-base/properties.h>
#include <android-base/stringprintf.h>
#include <android-base/strings.h>

namespace aidl {
namespace com {
namespace ftd {
namespace gyn {

ndk::ScopedAStatus GyndService::open(const std::string& path, int* _aidl_return) {
    std::lock_guard<std::mutex> guard(lock_);
    path_ = path;
    LOG(ERROR) << path_;

    LOG(ERROR) << "open path: " << path;

    // return fd to caller 
    *_aidl_return = 12;
    return ndk::ScopedAStatus::ok();
}

ndk::ScopedAStatus GyndService::close(int fd) {
    LOG(ERROR) << "Close device: " << fd;
    return ndk::ScopedAStatus::ok();
}

}  // namespace gyn
}  // namespace ftd
}  // namespace com
}  // namespace aidl

```


#### 10. default/include/gyn_service.h
```cpp
/*
 * Copyright (C) 2019 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
#pragma once

#include <map>
#include <memory>
#include <mutex>
#include <sstream>
#include <string>
#include <vector>

#include <android-base/unique_fd.h>
#include <aidl/com/ftd/gyn/BnGyndService.h>

namespace aidl {
namespace com {
namespace ftd {
namespace gyn {

static constexpr char kGyndServiceName[] = "gyndservice";

class GyndService : public BnGyndService {
  public:
    ndk::ScopedAStatus open(const std::string& path, int* _aidl_return) override;
    ndk::ScopedAStatus close(int fd) override;

  private:
    std::string path_ = {};
    std::mutex lock_;
    std::mutex& lock() { return lock_; }
};

}  // namespace gyn
}  // namespace ftd
}  // namespace com
}  // namespace aidl

```

### 添加权限

添加vendor侧sepolicy to BOARD_VENDOR_SEPOLICY_DIRS

对供应商代码可见的 AIDL 服务类型必须具有 vendor_service 属性。否则，sepolicy 配置将与任何其他 AIDL 服务相同。


#### 1. service.te

    type gynd_service, vendor_service, service_manager_type;

#### 2. service_contexts 

    com.ftd.gyn.IGyndService/default                       u:object_r:gynd_service:s0

#### 3. file_contexts 

    /vendor/bin/gynd                u:object_r:hal_gynd_default_exec:s0

#### 4. hal_gynd_default.te 

    # gynd - add for hal vendor service

    type hal_gynd_default, domain;

    type hal_gynd_default_exec, exec_type, vendor_file_type, file_type;

    init_daemon_domain(hal_gynd_default)

    #binder_use(hal_gynd_default)
    #add_service(hal_gynd_default, gynd_service)


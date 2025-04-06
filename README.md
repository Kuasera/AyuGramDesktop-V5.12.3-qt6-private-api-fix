
# AyuGram V5.12.3 QT6 Compatibility Fix

This repository provides a fix for the QT6 issue when building AyuGram Desktop on Linux (CachyOS, Arch Linux, etc.)

## 🚑 Problem

When building AyuGram with QT6 (especially version 6.9 and above), you may encounter this error:

```
    ayugram-desktop: symbol lookup error: ayugram-desktop: undefined symbol: _ZTI20QGenericUnixServices, version Qt_6_PRIVATE_API
```

## 🔍 Cause

This issue occurs because AyuGram's code expects an outdated QT service (`QGenericUnixServices`) which has been replaced by `QDesktopUnixServices` in QT 6.9 and above.

---

## 💡 Solution

### 1. Modify `base_linux_xdp_utilities.cpp`

Locate the file:

```
Telegram/lib_base/base/platform/linux/base_linux_xdp_utilities.cpp
```

Replace this line:

```cpp
#include <private/qgenericunixservices_p.h>
```

With:

```cpp
#if QT_VERSION >= QT_VERSION_CHECK(6, 9, 0)
#include <private/qdesktopunixservices_p.h>
#else
#include <private/qgenericunixservices_p.h>
#endif // Qt >= 6.9.0
```

---

### 2. Update Services Handling

Find this line:

```cpp
if (const auto services = dynamic_cast<QGenericUnixServices*>(
```

Replace it with:

```cpp
#if QT_VERSION >= QT_VERSION_CHECK(6, 9, 0)
if (const auto services = dynamic_cast<QDesktopUnixServices*>(
#else
if (const auto services = dynamic_cast<QGenericUnixServices*>(
#endif // Qt >= 6.9.0
```
---

###  Full code of base_linux_xdp_utilities.cpp (modified)
```cpp
// This file is part of Desktop App Toolkit,
// a set of libraries for developing nice desktop applications.
//
// For license and copyright information please follow this link:
// https://github.com/desktop-app/legal/blob/master/LEGAL
//
#include "base/platform/linux/base_linux_xdp_utilities.h"

#include "base/platform/base_platform_info.h"

#include <xdpsettings/xdpsettings.hpp>

#include <QtGui/QGuiApplication>
#include <QtGui/QWindow>

#if QT_VERSION >= QT_VERSION_CHECK(6, 5, 0)
#include <qpa/qplatformintegration.h>
#include <private/qguiapplication_p.h>

#if QT_VERSION >= QT_VERSION_CHECK(6, 9, 0)
#include <private/qdesktopunixservices_p.h>
#else
#include <private/qgenericunixservices_p.h>
#endif // Qt >= 6.9.0

#endif // Qt >= 6.5.0

#include <sstream>

namespace base::Platform::XDP {
namespace {

using namespace gi::repository;
namespace GObject = gi::repository::GObject;

} // namespace

std::string ParentWindowID() {
	return ParentWindowID(QGuiApplication::focusWindow());
}

std::string ParentWindowID(QWindow *window) {
	if (!window) {
		return {};
	}

#if QT_VERSION >= QT_VERSION_CHECK(6, 5, 0)

#if QT_VERSION >= QT_VERSION_CHECK(6, 9, 0)
	if (const auto services = dynamic_cast<QDesktopUnixServices*>(
#else
	if (const auto services = dynamic_cast<QGenericUnixServices*>(
#endif // Qt >= 6.9.0

			QGuiApplicationPrivate::platformIntegration()->services())) {
		return services->portalWindowIdentifier(window).toStdString();
	}
#endif // Qt >= 6.5.0

	if (::Platform::IsX11()) {
		std::stringstream result;
		result << "x11:" << std::hex << window->winId();
		return result.str();
	}

	return {};
}

gi::result<GLib::Variant> ReadSetting(
		const std::string &group,
		const std::string &key) {
	auto interface = gi::result<XdpSettings::Settings>(
		XdpSettings::SettingsProxy::new_for_bus_sync(
			Gio::BusType::SESSION_,
			Gio::DBusProxyFlags::NONE_,
			kService,
			kObjectPath));

	if (!interface) {
		return make_unexpected(std::move(interface.error()));
	}

	auto result = interface->call_read_one_sync(group, key);
	if (!result) {
		return make_unexpected(std::move(result.error()));
	}

	return std::get<1>(*result).get_variant();
}

class SettingWatcher::Private {
public:
	XdpSettings::Settings interface;
};

SettingWatcher::SettingWatcher(
		Fn<void(
			const std::string &,
			const std::string &,
			GLib::Variant)> callback)
: _private(std::make_unique<Private>()) {
	XdpSettings::SettingsProxy::new_for_bus(
		Gio::BusType::SESSION_,
		Gio::DBusProxyFlags::NONE_,
		kService,
		kObjectPath,
		crl::guard(this, [=](GObject::Object, Gio::AsyncResult res) {
			_private->interface = XdpSettings::Settings(
				XdpSettings::SettingsProxy::new_for_bus_finish(res, nullptr));

			if (!_private->interface) {
				return;
			}

			_private->interface.signal_setting_changed().connect([=](
					XdpSettings::Settings,
					std::string group,
					std::string key,
					GLib::Variant value) {
				callback(group, key, value.get_variant());
			});
		}));
}

SettingWatcher::~SettingWatcher() = default;

} // namespace base::Platform::XDP

```
---

### 3. Rebuild the Project

After applying the patch, rebuild AyuGram:

```bash
mkdir build
cd build
cmake .. -D TDESKTOP_API_ID=YOUR_API_ID -D TDESKTOP_API_HASH=YOUR_API_HASH
make -j$(nproc)
```

Replace `YOUR_API_ID` and `YOUR_API_HASH` with your own credentials from [Telegram API](https://my.telegram.org/auth).

---

### 4. Run the Application

```bash
./ayugram-desktop
```

---

## 🎉 Success

Your AyuGram should now work properly with QT 6.9 and above.

---

## 🤝 Contributing

If you find more issues or improvements, feel free to open an issue or submit a pull request.

---

## 📜 License

This solution is open-source and free to use. Feel free to modify and distribute it under your own terms.

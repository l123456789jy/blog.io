---
layout:     post
title:      "HUAWEI HiCar 有线连接"
subtitle:   ""
date:       2021-10-19 16:42:00
author:     ""
header-img: "img/home-bg-o.jpg"
header-mask: 0.3
catalog:
tags: Android
    -
---


- 根据文档，更改配置文件，和给需要的so以及apk提交系统签名，必须要系统权限

- 修改FW层架增加增加路由表


```
/*
 * Copyright (C) 2008 The Android Open Source Project
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

#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#define LOG_TAG "Netd"

#include <cutils/log.h>

#include <netutils/ifc.h>
#include <sysutils/NetlinkEvent.h>
#include "NetlinkHandler.h"
#include "NetlinkManager.h"
#include "ResponseCode.h"
#include "SockDiag.h"
#include "Controllers.h"
#include "RouteController.h"
static const char *kUpdated = "updated";
static const char *kRemoved = "removed";

namespace android {
namespace net {

NetlinkHandler::NetlinkHandler(NetlinkManager *nm, int listenerSocket,
                               int format) :
                        NetlinkListener(listenerSocket, format) {
    mNm = nm;
}

NetlinkHandler::~NetlinkHandler() {
}

int NetlinkHandler::start() {
    return this->startListener();
}

int NetlinkHandler::stop() {
    return this->stopListener();
}

static long parseIfIndex(const char* ifIndex) {
    if (ifIndex == nullptr) {
        return 0;
    }
    long ifaceIndex = strtol(ifIndex, NULL, 10);
    // strtol returns 0 on error, which is fine because 0 is not a valid ifindex.
    if (errno == ERANGE && (ifaceIndex == LONG_MAX || ifaceIndex == LONG_MIN)) {
        return 0;
    }
    return ifaceIndex;
}

void NetlinkHandler::onEvent(NetlinkEvent *evt) {
    const char *subsys = evt->getSubsystem();
    if (!subsys) {
        ALOGW("No subsystem found in netlink event");
        return;
    }

    if (!strcmp(subsys, "net")) {
        NetlinkEvent::Action action = evt->getAction();
        const char *iface = evt->findParam("INTERFACE");
        if ((action == NetlinkEvent::Action::kAdd) ||
            (action == NetlinkEvent::Action::kLinkUp) ||
            (action == NetlinkEvent::Action::kLinkDown)) {
            const char *ifIndex = evt->findParam("IFINDEX");
            long ifaceIndex = parseIfIndex(ifIndex);
            if (ifaceIndex) {
                gCtls->trafficCtrl.addInterface(iface, ifaceIndex);
            } else {
                ALOGE("invalid interface index: %s(%s)", iface, ifIndex);
            }
        }

        if (action == NetlinkEvent::Action::kAdd) {
            notifyInterfaceAdded(iface);
        } else if (action == NetlinkEvent::Action::kRemove) {
            notifyInterfaceRemoved(iface);
        } else if (action == NetlinkEvent::Action::kChange) {
            evt->dump();
            notifyInterfaceChanged("nana", true);
        } else if (action == NetlinkEvent::Action::kLinkUp) {
			     const char *iface = evt->findParam("INTERFACE");
                 if (!strncmp(iface, "wwan0", 5))
				 { 
				    int ret =RouteController::addRoute(iface, "192.168.66.0/24", NULL, RouteController::LEGACY_SYSTEM);
					 ALOGE("Error destroying sockets: %s", strerror(-ret));
				  
				 }
            notifyInterfaceLinkChanged(iface, true);
        } else if (action == NetlinkEvent::Action::kLinkDown) {
            notifyInterfaceLinkChanged(iface, false);
        } else if (action == NetlinkEvent::Action::kAddressUpdated ||
                   action == NetlinkEvent::Action::kAddressRemoved) {
            const char *address = evt->findParam("ADDRESS");
            const char *flags = evt->findParam("FLAGS");
            const char *scope = evt->findParam("SCOPE");
            const char *ifIndex = evt->findParam("IFINDEX");
            char addrstr[INET6_ADDRSTRLEN + strlen("/128")];
            strlcpy(addrstr, address, sizeof(addrstr));
            char *slash = strchr(addrstr, '/');
            if (slash) {
                *slash = '\0';
            }

            long ifaceIndex = parseIfIndex(ifIndex);
            if (!ifaceIndex) {
                ALOGE("invalid interface index: %s(%s)", iface, ifIndex);
            }
            if (action == NetlinkEvent::Action::kAddressUpdated) {
                gCtls->netCtrl.addInterfaceAddress(ifaceIndex, address);
            } else {  // action == NetlinkEvent::Action::kAddressRemoved
                bool shouldDestroy = gCtls->netCtrl.removeInterfaceAddress(ifaceIndex, address);
                if (shouldDestroy) {
                    SockDiag sd;
                    if (sd.open()) {
                        int ret = sd.destroySockets(addrstr);
                        if (ret < 0) {
                            ALOGE("Error destroying sockets: %s", strerror(-ret));
                        }
                    } else {
                        ALOGE("Error opening NETLINK_SOCK_DIAG socket: %s", strerror(errno));
                    }
                }
            }
            // Note: if this interface was deleted, iface is "" and we don't notify.
            if (iface && iface[0] && address && flags && scope) {
                notifyAddressChanged(action, address, iface, flags, scope);
            }
        } else if (action == NetlinkEvent::Action::kRdnss) {
            const char *lifetime = evt->findParam("LIFETIME");
            const char *servers = evt->findParam("SERVERS");
            if (lifetime && servers) {
                notifyInterfaceDnsServers(iface, lifetime, servers);
            }
        } else if (action == NetlinkEvent::Action::kRouteUpdated ||
                   action == NetlinkEvent::Action::kRouteRemoved) {
            const char *route = evt->findParam("ROUTE");
            const char *gateway = evt->findParam("GATEWAY");
            const char *iface = evt->findParam("INTERFACE");
            if (route && (gateway || iface)) {
                notifyRouteChange(action, route, gateway, iface);
            }
        }

    } else if (!strcmp(subsys, "qlog") || !strcmp(subsys, "xt_quota2")) {
        const char *alertName = evt->findParam("ALERT_NAME");
        const char *iface = evt->findParam("INTERFACE");
        notifyQuotaLimitReached(alertName, iface);

    } else if (!strcmp(subsys, "strict")) {
        const char *uid = evt->findParam("UID");
        const char *hex = evt->findParam("HEX");
        notifyStrictCleartext(uid, hex);

    } else if (!strcmp(subsys, "xt_idletimer")) {
        const char *label = evt->findParam("INTERFACE");
        const char *state = evt->findParam("STATE");
        const char *timestamp = evt->findParam("TIME_NS");
        const char *uid = evt->findParam("UID");
        if (state)
            notifyInterfaceClassActivity(label, !strcmp("active", state),
                                         timestamp, uid);

#if !LOG_NDEBUG
    } else if (strcmp(subsys, "platform") && strcmp(subsys, "backlight")) {
        /* It is not a VSYNC or a backlight event */
        ALOGV("unexpected event from subsystem %s", subsys);
#endif
    }
}

void NetlinkHandler::notify(int code, const char *format, ...) {
    char *msg;
    va_list args;
    va_start(args, format);
    if (vasprintf(&msg, format, args) >= 0) {
        mNm->getBroadcaster()->sendBroadcast(code, msg, false);
        free(msg);
    } else {
        SLOGE("Failed to send notification: vasprintf: %s", strerror(errno));
    }
    va_end(args);
}

void NetlinkHandler::notifyInterfaceAdded(const char *name) {
    notify(ResponseCode::InterfaceChange, "Iface added %s", name);
}

void NetlinkHandler::notifyInterfaceRemoved(const char *name) {
    notify(ResponseCode::InterfaceChange, "Iface removed %s", name);
}

void NetlinkHandler::notifyInterfaceChanged(const char *name, bool isUp) {
    notify(ResponseCode::InterfaceChange,
           "Iface changed %s %s", name, (isUp ? "up" : "down"));
}

void NetlinkHandler::notifyInterfaceLinkChanged(const char *name, bool isUp) {
    notify(ResponseCode::InterfaceChange,
           "Iface linkstate %s %s", name, (isUp ? "up" : "down"));
}

void NetlinkHandler::notifyQuotaLimitReached(const char *name, const char *iface) {
    notify(ResponseCode::BandwidthControl, "limit alert %s %s", name, iface);
}

void NetlinkHandler::notifyInterfaceClassActivity(const char *name,
                                                  bool isActive,
                                                  const char *timestamp,
                                                  const char *uid) {
    if (timestamp == NULL)
        notify(ResponseCode::InterfaceClassActivity,
           "IfaceClass %s %s", isActive ? "active" : "idle", name);
    else if (uid != NULL && isActive)
        notify(ResponseCode::InterfaceClassActivity,
           "IfaceClass active %s %s %s", name, timestamp, uid);
    else
        notify(ResponseCode::InterfaceClassActivity,
           "IfaceClass %s %s %s", isActive ? "active" : "idle", name, timestamp);
}

void NetlinkHandler::notifyAddressChanged(NetlinkEvent::Action action, const char *addr,
                                          const char *iface, const char *flags,
                                          const char *scope) {
    notify(ResponseCode::InterfaceAddressChange,
           "Address %s %s %s %s %s",
           (action == NetlinkEvent::Action::kAddressUpdated) ? kUpdated : kRemoved,
           addr, iface, flags, scope);
}

void NetlinkHandler::notifyInterfaceDnsServers(const char *iface,
                                               const char *lifetime,
                                               const char *servers) {
    notify(ResponseCode::InterfaceDnsInfo, "DnsInfo servers %s %s %s",
           iface, lifetime, servers);
}

void NetlinkHandler::notifyRouteChange(NetlinkEvent::Action action, const char *route,
                                       const char *gateway, const char *iface) {
    notify(ResponseCode::RouteChange,
           "Route %s %s%s%s%s%s",
           (action == NetlinkEvent::Action::kRouteUpdated) ? kUpdated : kRemoved,
           route,
           (gateway && *gateway) ? " via " : "",
           gateway,
           (iface && *iface) ? " dev " : "",
           iface);
}

void NetlinkHandler::notifyStrictCleartext(const char* uid, const char* hex) {
    notify(ResponseCode::StrictCleartext, "%s %s", uid, hex);
}

}  // namespace net
}  // namespace android

```

- 以上代码修改 system/netd/server 下的 NetlinkHandler.cpp  文件直接替换即可
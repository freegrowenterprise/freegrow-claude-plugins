---
name: ble-development-patterns
description: >
  Bluetooth Low Energy development patterns including GATT design, advertising strategies,
  connection optimization, and BLE security. Use when implementing BLE peripherals or centrals.
---

# BLE Development Patterns

Comprehensive guide to Bluetooth Low Energy embedded development.

## When to Use

- Designing BLE GATT services
- Implementing BLE peripherals or centrals
- Optimizing BLE power consumption
- Implementing BLE security and bonding
- Troubleshooting BLE connectivity issues

## BLE Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                   Application                        │
├─────────────────────────────────────────────────────┤
│                      GAP                             │
│    (Advertising, Scanning, Connection Management)   │
├─────────────────────────────────────────────────────┤
│                     GATT                             │
│        (Services, Characteristics, Profiles)        │
├─────────────────────────────────────────────────────┤
│                      ATT                             │
│           (Attribute Protocol)                       │
├─────────────────────────────────────────────────────┤
│                      L2CAP                           │
│     (Logical Link Control and Adaptation)           │
├─────────────────────────────────────────────────────┤
│                   Link Layer                         │
│      (Packets, Channels, State Machine)             │
├─────────────────────────────────────────────────────┤
│                Physical Layer                        │
│           (2.4 GHz Radio, 40 Channels)              │
└─────────────────────────────────────────────────────┘
```

## GATT Service Design

### Service Structure

```c
/* Well-designed GATT service structure */

/*
 * Custom Sensor Service (UUID: 0x1234...)
 * ├── Sensor Data Characteristic (UUID: 0x1235...)
 * │   ├── Properties: Read, Notify
 * │   ├── Value: [temperature:int16, humidity:uint16, pressure:uint32]
 * │   └── CCCD: Client Characteristic Configuration
 * ├── Configuration Characteristic (UUID: 0x1236...)
 * │   ├── Properties: Read, Write
 * │   └── Value: [sample_rate:uint8, enabled:bool]
 * └── Control Point Characteristic (UUID: 0x1237...)
 *     ├── Properties: Write, Indicate
 *     └── Value: [command:uint8, params:uint8[4]]
 */

/* UUID definitions */
#define UUID_SENSOR_SERVICE     0x12340000, 0x1234, 0x1234, 0x1234, 0x123456789abc
#define UUID_SENSOR_DATA        0x12350000, 0x1234, 0x1234, 0x1234, 0x123456789abc
#define UUID_SENSOR_CONFIG      0x12360000, 0x1234, 0x1234, 0x1234, 0x123456789abc
#define UUID_SENSOR_CONTROL     0x12370000, 0x1234, 0x1234, 0x1234, 0x123456789abc

/* Data structures - pack for BLE transmission */
typedef struct __attribute__((packed)) {
    int16_t temperature;    /* 0.01°C units */
    uint16_t humidity;      /* 0.01% units */
    uint32_t pressure;      /* Pa units */
    uint32_t timestamp;     /* Unix timestamp */
} sensor_data_t;

typedef struct __attribute__((packed)) {
    uint8_t sample_rate;    /* Samples per second */
    uint8_t enabled;        /* 0=off, 1=on */
    uint8_t resolution;     /* Sensor resolution setting */
} sensor_config_t;

typedef struct __attribute__((packed)) {
    uint8_t opcode;
    uint8_t params[4];
} control_point_t;

/* Control point opcodes */
enum control_opcode {
    CTRL_RESET = 0x01,
    CTRL_CALIBRATE = 0x02,
    CTRL_START_STREAM = 0x03,
    CTRL_STOP_STREAM = 0x04,
};
```

### GATT Server Implementation Pattern

```c
/* GATT service with proper error handling */

static sensor_data_t sensor_data;
static sensor_config_t sensor_config = {
    .sample_rate = 1,
    .enabled = 1,
    .resolution = 2,
};

/* Read callback with validation */
static ssize_t read_sensor_data(struct bt_conn *conn,
                                const struct bt_gatt_attr *attr,
                                void *buf, uint16_t len, uint16_t offset)
{
    /* Refresh data before read */
    update_sensor_data(&sensor_data);

    return bt_gatt_attr_read(conn, attr, buf, len, offset,
                             &sensor_data, sizeof(sensor_data));
}

/* Write callback with validation and response */
static ssize_t write_sensor_config(struct bt_conn *conn,
                                   const struct bt_gatt_attr *attr,
                                   const void *buf, uint16_t len,
                                   uint16_t offset, uint8_t flags)
{
    /* Validate length */
    if (len != sizeof(sensor_config_t)) {
        return BT_GATT_ERR(BT_ATT_ERR_INVALID_ATTRIBUTE_LEN);
    }

    /* Validate offset */
    if (offset != 0) {
        return BT_GATT_ERR(BT_ATT_ERR_INVALID_OFFSET);
    }

    const sensor_config_t *new_config = buf;

    /* Validate values */
    if (new_config->sample_rate < 1 || new_config->sample_rate > 100) {
        return BT_GATT_ERR(BT_ATT_ERR_VALUE_NOT_ALLOWED);
    }

    if (new_config->resolution > 3) {
        return BT_GATT_ERR(BT_ATT_ERR_VALUE_NOT_ALLOWED);
    }

    /* Apply configuration */
    memcpy(&sensor_config, new_config, sizeof(sensor_config_t));
    apply_sensor_config(&sensor_config);

    return len;
}

/* Control point with indication response */
static ssize_t write_control_point(struct bt_conn *conn,
                                   const struct bt_gatt_attr *attr,
                                   const void *buf, uint16_t len,
                                   uint16_t offset, uint8_t flags)
{
    if (len < 1) {
        return BT_GATT_ERR(BT_ATT_ERR_INVALID_ATTRIBUTE_LEN);
    }

    const control_point_t *cmd = buf;
    uint8_t response[2] = {cmd->opcode, 0x00};  /* opcode + status */

    switch (cmd->opcode) {
    case CTRL_RESET:
        reset_sensor();
        response[1] = 0x01;  /* Success */
        break;

    case CTRL_CALIBRATE:
        if (calibrate_sensor() == 0) {
            response[1] = 0x01;
        } else {
            response[1] = 0x03;  /* Operation failed */
        }
        break;

    default:
        response[1] = 0x02;  /* Opcode not supported */
    }

    /* Send indication response */
    bt_gatt_notify(conn, attr, response, sizeof(response));

    return len;
}
```

## Advertising Strategies

### Advertising Types and Use Cases

| Type | Use Case | Connectable | Power |
|------|----------|-------------|-------|
| **ADV_IND** | General peripheral | Yes | Medium |
| **ADV_DIRECT_IND** | Fast reconnect | Yes (specific) | High |
| **ADV_NONCONN_IND** | Beacon | No | Low |
| **ADV_SCAN_IND** | Scannable beacon | No | Low |
| **Extended ADV** | Long data, coded PHY | Configurable | Variable |

### Advertising Data Optimization

```c
/* Maximum advertising data: 31 bytes (legacy) or 254 bytes (extended) */

/* Efficient advertising data layout */
static const struct bt_data ad[] = {
    /* Flags - Required (3 bytes) */
    BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),

    /* Complete list of 16-bit Service UUIDs (4 bytes for one UUID) */
    BT_DATA_BYTES(BT_DATA_UUID16_ALL,
                  BT_UUID_16_ENCODE(BT_UUID_HRS_VAL),
                  BT_UUID_16_ENCODE(BT_UUID_BAS_VAL)),

    /* Manufacturer Specific Data (variable) */
    BT_DATA_BYTES(BT_DATA_MANUFACTURER_DATA,
                  0xFF, 0xFF,  /* Company ID (0xFFFF = testing) */
                  0x01, 0x02, 0x03),  /* Custom data */

    /* TX Power Level (3 bytes) - helps with distance estimation */
    BT_DATA_BYTES(BT_DATA_TX_POWER, 0x00),  /* 0 dBm */
};

/* Scan response data - sent on request */
static const struct bt_data sd[] = {
    /* Complete Local Name */
    BT_DATA(BT_DATA_NAME_COMPLETE, DEVICE_NAME, sizeof(DEVICE_NAME) - 1),

    /* Service Data */
    BT_DATA_BYTES(BT_DATA_SVC_DATA16,
                  BT_UUID_16_ENCODE(BT_UUID_HRS_VAL),
                  0x50),  /* Heart rate: 80 BPM */
};

/* Advertising with service data update */
int update_advertising_data(uint8_t heart_rate)
{
    /* Create new service data */
    uint8_t svc_data[] = {
        BT_UUID_16_ENCODE(BT_UUID_HRS_VAL) >> 0,
        BT_UUID_16_ENCODE(BT_UUID_HRS_VAL) >> 8,
        heart_rate
    };

    struct bt_data new_sd[] = {
        BT_DATA(BT_DATA_NAME_COMPLETE, DEVICE_NAME, sizeof(DEVICE_NAME) - 1),
        BT_DATA(BT_DATA_SVC_DATA16, svc_data, sizeof(svc_data)),
    };

    return bt_le_adv_update_data(ad, ARRAY_SIZE(ad), new_sd, ARRAY_SIZE(new_sd));
}
```

### Advertising Interval Selection

```c
/* Advertising interval affects discovery time and power */

/* Fast advertising for quick connection (high power) */
#define ADV_FAST_INT_MIN 32   /* 20ms (32 * 0.625ms) */
#define ADV_FAST_INT_MAX 48   /* 30ms */

/* Slow advertising for background presence (low power) */
#define ADV_SLOW_INT_MIN 1600 /* 1000ms */
#define ADV_SLOW_INT_MAX 1600 /* 1000ms */

/* Beacon advertising (very low power) */
#define ADV_BEACON_INT 16000  /* 10s */

/* Start with fast advertising, switch to slow after timeout */
static struct k_work_delayable adv_slow_work;

void switch_to_slow_advertising(struct k_work *work)
{
    bt_le_adv_stop();

    struct bt_le_adv_param slow_param = {
        .id = BT_ID_DEFAULT,
        .options = BT_LE_ADV_OPT_CONNECTABLE,
        .interval_min = ADV_SLOW_INT_MIN,
        .interval_max = ADV_SLOW_INT_MAX,
    };

    bt_le_adv_start(&slow_param, ad, ARRAY_SIZE(ad), sd, ARRAY_SIZE(sd));
}

void start_advertising(void)
{
    struct bt_le_adv_param fast_param = {
        .id = BT_ID_DEFAULT,
        .options = BT_LE_ADV_OPT_CONNECTABLE,
        .interval_min = ADV_FAST_INT_MIN,
        .interval_max = ADV_FAST_INT_MAX,
    };

    bt_le_adv_start(&fast_param, ad, ARRAY_SIZE(ad), sd, ARRAY_SIZE(sd));

    /* Switch to slow advertising after 30 seconds */
    k_work_schedule(&adv_slow_work, K_SECONDS(30));
}
```

## Connection Parameter Optimization

### Parameter Selection Guide

| Use Case | Interval | Latency | Timeout | Power |
|----------|----------|---------|---------|-------|
| High throughput | 7.5-15ms | 0 | 2s | High |
| Balanced | 30-50ms | 0-2 | 4s | Medium |
| Low power | 100-500ms | 4-10 | 10s | Low |
| Ultra low power | 1-2s | 10-30 | 30s | Minimal |

```c
/* Connection parameter negotiation */

struct conn_param_config {
    uint16_t interval_min;
    uint16_t interval_max;
    uint16_t latency;
    uint16_t timeout;
};

/* Different profiles for different application states */
static const struct conn_param_config conn_params[] = {
    [CONN_PARAM_HIGH_THROUGHPUT] = {
        .interval_min = 6,    /* 7.5ms */
        .interval_max = 6,
        .latency = 0,
        .timeout = 200,       /* 2s */
    },
    [CONN_PARAM_BALANCED] = {
        .interval_min = 24,   /* 30ms */
        .interval_max = 40,   /* 50ms */
        .latency = 2,
        .timeout = 400,       /* 4s */
    },
    [CONN_PARAM_LOW_POWER] = {
        .interval_min = 80,   /* 100ms */
        .interval_max = 400,  /* 500ms */
        .latency = 4,
        .timeout = 600,       /* 6s */
    },
};

int request_conn_params(struct bt_conn *conn, enum conn_param_profile profile)
{
    const struct conn_param_config *cfg = &conn_params[profile];

    struct bt_le_conn_param param = {
        .interval_min = cfg->interval_min,
        .interval_max = cfg->interval_max,
        .latency = cfg->latency,
        .timeout = cfg->timeout,
    };

    return bt_conn_le_param_update(conn, &param);
}
```

## BLE Security

### Pairing and Bonding

```c
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/conn.h>

/* Security callbacks */
static void auth_passkey_display(struct bt_conn *conn, unsigned int passkey)
{
    char addr[BT_ADDR_LE_STR_LEN];
    bt_addr_le_to_str(bt_conn_get_dst(conn), addr, sizeof(addr));
    printk("Passkey for %s: %06u\n", addr, passkey);
}

static void auth_passkey_confirm(struct bt_conn *conn, unsigned int passkey)
{
    char addr[BT_ADDR_LE_STR_LEN];
    bt_addr_le_to_str(bt_conn_get_dst(conn), addr, sizeof(addr));
    printk("Confirm passkey %06u for %s? (y/n)\n", passkey, addr);
    /* In real app, get user confirmation */
    bt_conn_auth_passkey_confirm(conn);
}

static void auth_cancel(struct bt_conn *conn)
{
    printk("Pairing cancelled\n");
}

static void pairing_complete(struct bt_conn *conn, bool bonded)
{
    printk("Pairing %s\n", bonded ? "complete (bonded)" : "complete");
}

static void pairing_failed(struct bt_conn *conn, enum bt_security_err reason)
{
    printk("Pairing failed: %d\n", reason);
}

static struct bt_conn_auth_cb auth_callbacks = {
    .passkey_display = auth_passkey_display,
    .passkey_confirm = auth_passkey_confirm,
    .cancel = auth_cancel,
};

static struct bt_conn_auth_info_cb auth_info_callbacks = {
    .pairing_complete = pairing_complete,
    .pairing_failed = pairing_failed,
};

/* Initialize security */
void init_security(void)
{
    bt_conn_auth_cb_register(&auth_callbacks);
    bt_conn_auth_info_cb_register(&auth_info_callbacks);
}

/* Request security level */
void request_security(struct bt_conn *conn)
{
    int err = bt_conn_set_security(conn, BT_SECURITY_L2);
    if (err) {
        printk("Failed to set security: %d\n", err);
    }
}
```

### Security Levels

```c
/* BLE Security Levels */
/*
 * Level 1: No security (no authentication, no encryption)
 * Level 2: Unauthenticated encryption (Just Works pairing)
 * Level 3: Authenticated encryption (Passkey or OOB)
 * Level 4: Authenticated LE Secure Connections (LESC)
 */

/* Characteristic with security requirement */
BT_GATT_CHARACTERISTIC(BT_UUID_CUSTOM_CHAR,
    BT_GATT_CHRC_READ | BT_GATT_CHRC_WRITE,
    BT_GATT_PERM_READ_ENCRYPT |     /* Requires encryption */
    BT_GATT_PERM_WRITE_AUTHEN,      /* Requires authentication */
    read_handler, write_handler, &value),
```

## Data Throughput Optimization

### Maximum Throughput Configuration

```c
/* Maximize BLE throughput */

/* 1. Use maximum MTU */
#define MAX_MTU 247  /* BLE 4.2+ */

/* prj.conf */
// CONFIG_BT_L2CAP_TX_MTU=247
// CONFIG_BT_BUF_ACL_TX_SIZE=251
// CONFIG_BT_BUF_ACL_RX_SIZE=251

/* 2. Use Data Length Extension */
// CONFIG_BT_CTLR_DATA_LENGTH_MAX=251

/* 3. Use 2M PHY */
void request_phy_update(struct bt_conn *conn)
{
    struct bt_conn_le_phy_param phy = {
        .options = BT_CONN_LE_PHY_OPT_NONE,
        .pref_tx_phy = BT_GAP_LE_PHY_2M,
        .pref_rx_phy = BT_GAP_LE_PHY_2M,
    };
    bt_conn_le_phy_update(conn, &phy);
}

/* 4. Queue multiple notifications */
#define NOTIFY_QUEUE_SIZE 10

int send_data_stream(struct bt_conn *conn, const uint8_t *data, size_t len)
{
    size_t mtu = bt_gatt_get_mtu(conn) - 3;  /* ATT header */
    size_t offset = 0;
    int queued = 0;

    while (offset < len && queued < NOTIFY_QUEUE_SIZE) {
        size_t chunk = MIN(mtu, len - offset);

        int err = bt_gatt_notify(conn, &svc.attrs[1], data + offset, chunk);
        if (err == -ENOMEM) {
            /* Queue full, wait and retry */
            k_sleep(K_MSEC(10));
            continue;
        }
        if (err) {
            return err;
        }

        offset += chunk;
        queued++;
    }

    return 0;
}
```

## Debugging BLE Issues

### Common Issues and Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| Device not discoverable | Not in scan results | Check advertising params, TX power |
| Connection drops | Frequent disconnects | Increase supervision timeout |
| Slow data transfer | Low throughput | Increase MTU, use 2M PHY |
| Pairing fails | Security errors | Check IO capabilities match |
| Notifications not working | No data received | Verify CCCD written |

### Debug Logging

```kconfig
# prj.conf - Comprehensive BLE debug logging
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=4

CONFIG_BT_DEBUG_LOG=y
CONFIG_BT_DEBUG_HCI_CORE=y
CONFIG_BT_DEBUG_CONN=y
CONFIG_BT_DEBUG_GATT=y
CONFIG_BT_DEBUG_ATT=y
CONFIG_BT_DEBUG_L2CAP=y
CONFIG_BT_DEBUG_SMP=y
```

### Packet Analysis

```c
/* Log GATT operations for debugging */
static ssize_t debug_read(struct bt_conn *conn,
                          const struct bt_gatt_attr *attr,
                          void *buf, uint16_t len, uint16_t offset)
{
    char addr[BT_ADDR_LE_STR_LEN];
    bt_addr_le_to_str(bt_conn_get_dst(conn), addr, sizeof(addr));

    LOG_DBG("Read from %s, handle=0x%04x, offset=%u, len=%u",
            addr, attr->handle, offset, len);

    /* Actual read implementation */
    return bt_gatt_attr_read(conn, attr, buf, len, offset, value, value_len);
}
```

## Best Practices

### DO
- Design GATT services before implementation
- Use appropriate connection parameters for use case
- Implement proper error handling in callbacks
- Test with multiple phone models and OS versions
- Use bonding for returning devices

### DON'T
- Send notifications without checking CCCD
- Ignore MTU negotiation
- Use hardcoded connection parameters
- Skip security for sensitive data
- Assume BLE range is consistent

## Resources

- [Bluetooth Core Specification](https://www.bluetooth.com/specifications/specs/)
- [Zephyr Bluetooth Stack](https://docs.zephyrproject.org/latest/connectivity/bluetooth/)
- [Nordic Semiconductor Infocenter](https://infocenter.nordicsemi.com/)
- [Android BLE Best Practices](https://punchthrough.com/android-ble-guide/)
- [iOS Core Bluetooth](https://developer.apple.com/documentation/corebluetooth)

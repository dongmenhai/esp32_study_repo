# ESP32C3/ ESP32S3 BLE 5.0 吞吐量测试

## 简介

当前 esp32s3 和 esp32c3 都支持 BLE5.0 协议，也就是支持 2M PHY 的通讯，基于官方 BLE 吞吐量测试 demo 进行修改，验证 2M PHY 的传输速率提升

主要参照 demo:  [ble_throughput](https://github.com/espressif/esp-idf/tree/master/examples/bluetooth/bluedroid/ble/ble_throughput)  client 和 server、[ble50_security_client](https://github.com/espressif/esp-idf/tree/master/examples/bluetooth/bluedroid/ble_50/ble50_security_client)  和 [ble50_security_server](https://github.com/espressif/esp-idf/tree/master/examples/bluetooth/bluedroid/ble_50/ble50_security_server)

## 修改

### throughput_server

throughput_server 应当改成 BLE5.0 的扩展广播，增加以下扩展广播数据与参数的配置，可以修改下 adv name

```c
#define EXT_ADV_HANDLE                            0
#define NUM_EXT_ADV_SET                           1
#define EXT_ADV_DURATION                          0
#define EXT_ADV_MAX_EVENTS                        0

static uint8_t ext_adv_raw_data[] = {
        0x02, 0x01, 0x06,
        0x02, 0x0a, 0xeb, 0x03, 0x03, 0xab, 0xcd,
        0x13, 0X09, 'T', 'H', 'R', 'O', 'U', 'G', 'H', 'P', 'U', 'T', '_', 'D', 'E', 'M', 'O', '_', '5', '0',
};

static esp_ble_gap_ext_adv_t ext_adv[1] = {
    [0] = {EXT_ADV_HANDLE, EXT_ADV_DURATION, EXT_ADV_MAX_EVENTS},
};

esp_ble_gap_ext_adv_params_t ext_adv_params_2M = {
    .type = ESP_BLE_GAP_SET_EXT_ADV_PROP_CONNECTABLE,
    .interval_min = 0x20,
    .interval_max = 0x20,
    .channel_map = ADV_CHNL_ALL,
    .filter_policy = ADV_FILTER_ALLOW_SCAN_ANY_CON_ANY,
    .primary_phy = ESP_BLE_GAP_PHY_1M,
    .max_skip = 0,
    .secondary_phy = ESP_BLE_GAP_PHY_2M,
    .sid = 0,
    .scan_req_notif = false,
    .own_addr_type = BLE_ADDR_TYPE_PUBLIC,
};
```

throughput_server 配置和启动广播是在 `gatts_profile_a_event_handler`  的 `ESP_GATTS_REG_EVT` 事件中开始的，这里使用 `esp_ble_gap_ext_adv_set_params` 开启扩展广播配置

```c
static void gatts_profile_a_event_handler(esp_gatts_cb_event_t event, esp_gatt_if_t gatts_if, esp_ble_gatts_cb_param_t *param) {
	switch (event) {
    case ESP_GATTS_REG_EVT:
		// ...
        esp_err_t set_dev_name_ret = esp_ble_gap_set_device_name(TEST_DEVICE_NAME);
        if (set_dev_name_ret){
            ESP_LOGE(GATTS_TAG, "set device name failed, error code = %x", set_dev_name_ret);
        }
        esp_ble_gap_ext_adv_set_params(EXT_ADV_HANDLE, &ext_adv_params_2M);
        esp_ble_gatts_create_service(gatts_if, &gl_profile_tab[PROFILE_A_APP_ID].service_id, GATTS_NUM_HANDLE_TEST_A);
        break;
    // ...
	}
}
```

在 `gap_event_handler`  回调中补充扩展广播配置参数、配置数据和开启广播完成事件

```c
static void gap_event_handler(esp_gap_ble_cb_event_t event, esp_ble_gap_cb_param_t *param)
{
    switch (event) {
    case ESP_GAP_BLE_EXT_ADV_SET_PARAMS_COMPLETE_EVT:
        ESP_LOGI(GATTS_TAG,"ESP_GAP_BLE_EXT_ADV_SET_PARAMS_COMPLETE_EVT status %d",  param->ext_adv_set_params.status);
        esp_ble_gap_config_ext_adv_data_raw(EXT_ADV_HANDLE,  sizeof(ext_adv_raw_data), &ext_adv_raw_data[0]);
        break;
    case ESP_GAP_BLE_EXT_ADV_DATA_SET_COMPLETE_EVT:
         ESP_LOGI(GATTS_TAG,"ESP_GAP_BLE_EXT_ADV_DATA_SET_COMPLETE_EVT status %d",  param->ext_adv_data_set.status);
         esp_ble_gap_ext_adv_start(NUM_EXT_ADV_SET, &ext_adv[0]);
         break;
    case ESP_GAP_BLE_EXT_ADV_START_COMPLETE_EVT:
         ESP_LOGI(GATTS_TAG, "ESP_GAP_BLE_EXT_ADV_START_COMPLETE_EVT, status = %d", param->ext_adv_data_set.status);
        break;
    //...
   }
}
```

同时也在建立连接事件 `ESP_GATTS_CONNECT_EVT` 中读取建立连接的 PHY ，来确定是否是建立的 2M PHY 的连接;  断连事件 `ESP_GATTS_DISCONNECT_EVT`  中

使用 `esp_ble_gap_ext_adv_start` 重新开启扩展广播

```c
static void gatts_profile_a_event_handler(esp_gatts_cb_event_t event, esp_gatt_if_t gatts_if, esp_ble_gatts_cb_param_t *param) {
	switch (event) 
    case ESP_GATTS_CONNECT_EVT: {
		// ...
        //start sent the update connection parameters to the peer device.
        esp_ble_gap_update_conn_params(&conn_params);
        esp_ble_gap_read_phy(param->connect.remote_bda);
        break;
    case ESP_GATTS_DISCONNECT_EVT:
        is_connect = false;
        ESP_LOGI(GATTS_TAG, "ESP_GATTS_DISCONNECT_EVT");
        esp_ble_gap_ext_adv_start(NUM_EXT_ADV_SET, &ext_adv[0]);
        break;
    //...
    }
}
static void gap_event_handler(esp_gap_ble_cb_event_t event, esp_ble_gap_cb_param_t *param)
{
    switch (event) {
    case ESP_GAP_BLE_READ_PHY_COMPLETE_EVT:
        ESP_LOGI(GATTS_TAG, "ESP_GAP_BLE_READ_PHY_COMPLETE_EVT, status = %d, tx_phy = %d, rx_phy = %d", 
            param->read_phy.status, param->read_phy.tx_phy, param->read_phy.rx_phy);
        esp_log_buffer_hex("read_phy bda:", param->read_phy.bda, 6);
        break;
    //...
   }
}
```

### throughput_client

throughput_client 应当开启扩展扫描，增加以下扩展参数的配置

```c
#define EXT_SCAN_DURATION     0
#define EXT_SCAN_PERIOD       0

static const char remote_device_name[] = "THROUGHPUT_DEMO_50";

static esp_ble_ext_scan_params_t ext_scan_params = {
    .own_addr_type = BLE_ADDR_TYPE_PUBLIC,
    .filter_policy = BLE_SCAN_FILTER_ALLOW_ALL,
    .scan_duplicate = BLE_SCAN_DUPLICATE_DISABLE,
    .cfg_mask = ESP_BLE_GAP_EXT_SCAN_CFG_UNCODE_MASK | ESP_BLE_GAP_EXT_SCAN_CFG_CODE_MASK,
    .uncoded_cfg = {BLE_SCAN_TYPE_ACTIVE, 40, 40},
    .coded_cfg = {BLE_SCAN_TYPE_ACTIVE, 40, 40},
};
```

throughput_client 的配置扫描参数是在 `gattc_profile_event_handler` 的 `ESP_GATTC_REG_EVT` 开始的，替换为扩展扫描参数配置

```c
static void gattc_profile_event_handler(esp_gattc_cb_event_t event, esp_gatt_if_t gattc_if, esp_ble_gattc_cb_param_t *param)
{
    esp_ble_gattc_cb_param_t *p_data = (esp_ble_gattc_cb_param_t *)param;

    switch (event) {
    case ESP_GATTC_REG_EVT:
        ESP_LOGI(GATTC_TAG, "REG_EVT");
        esp_err_t scan_ret = esp_ble_gap_set_ext_scan_params(&ext_scan_params);
        if (scan_ret){
            ESP_LOGE(GATTC_TAG, "set extend scan params error, error code = %x", scan_ret);
        }
        break;
    }
    // ...
}
```

同样在 `esp_gap_cb` 回调中配置扩展扫描开启及扫描到 server 设备的连接操作

```c
static void esp_gap_cb(esp_gap_ble_cb_event_t event, esp_ble_gap_cb_param_t *param)
{
    uint8_t *adv_name = NULL;
    uint8_t adv_name_len = 0;
    switch (event) {
    case ESP_GAP_BLE_SET_EXT_SCAN_PARAMS_COMPLETE_EVT: {
        if (param->set_ext_scan_params.status != ESP_BT_STATUS_SUCCESS) {
            ESP_LOGE(GATTC_TAG, "extend scan parameters set failed, error status = %x", param->set_ext_scan_params.status);
            break;
        }
        //the unit of the duration is second
        esp_ble_gap_start_ext_scan(EXT_SCAN_DURATION, EXT_SCAN_PERIOD);
        break;
    }
    case ESP_GAP_BLE_EXT_SCAN_START_COMPLETE_EVT:
        if (param->ext_scan_start.status != ESP_BT_STATUS_SUCCESS) {
            ESP_LOGE(GATTC_TAG, "scan start failed, error status = %x", param->scan_start_cmpl.status);
            break;
        }
        ESP_LOGI(GATTC_TAG, "Scan start success");
        break;
    case ESP_GAP_BLE_EXT_ADV_REPORT_EVT: {
        uint8_t *adv_name = NULL;
        uint8_t adv_name_len = 0;
        if(param->ext_adv_report.params.event_type & ESP_BLE_GAP_SET_EXT_ADV_PROP_LEGACY) {
            ESP_LOGI(GATTC_TAG, "legacy adv, adv type 0x%x data len %d", param->ext_adv_report.params.event_type, param->ext_adv_report.params.adv_data_len);
        } else {
            ESP_LOGI(GATTC_TAG, "extend adv, adv type 0x%x data len %d", param->ext_adv_report.params.event_type, param->ext_adv_report.params.adv_data_len);
        }
        adv_name = esp_ble_resolve_adv_data(param->ext_adv_report.params.adv_data,
                                            ESP_BLE_AD_TYPE_NAME_CMPL, &adv_name_len);
        if (!connect && strlen(remote_device_name) == adv_name_len && strncmp((char *)adv_name, remote_device_name, adv_name_len) == 0) {
            connect = true;
            esp_ble_gap_stop_ext_scan();
            esp_log_buffer_hex("adv addr", param->ext_adv_report.params.addr, 6);
            esp_log_buffer_char("adv name", adv_name, adv_name_len);
            ESP_LOGI(GATTC_TAG, "Stop extend scan and create aux open, primary_phy %d secondary phy %d\n", param->ext_adv_report.params.primary_phy, param->ext_adv_report.params.secondly_phy);

            esp_ble_gattc_aux_open(gl_profile_tab[PROFILE_A_APP_ID].gattc_if,
                                    param->ext_adv_report.params.addr,
                                    param->ext_adv_report.params.addr_type, true);
        }

        break;
    }
    case ESP_GAP_BLE_EXT_SCAN_STOP_COMPLETE_EVT:
        if (param->ext_scan_stop.status != ESP_BT_STATUS_SUCCESS){
            ESP_LOGE(GATTC_TAG, "extend Scan stop failed, error status = %x", param->ext_scan_stop.status);
            break;
        }
        ESP_LOGI(GATTC_TAG, "Stop extend scan successfully");
        break;
    //...
    }
}
```

### sdkconfig

在client 和 server demo 的 sdkconfig.defaults.esp32c3 和 sdkconfig.defaults.esp32s3 文件中增加 BLE5.0 feature 的使能，不然编译不过

```
CONFIG_BT_BLE_50_FEATURES_SUPPORTED=y
```

## 优化

对于 BLE 吞吐量测试，有两个因素对吞吐量比较大：

* 一是单包数据量， 官方 demo 给的应用层 ATT packet 的长度是 490，可能是因为在链路层可以分两包发出去，如果把 ATT packet 的长度改成 512， 那就要分 3 包发出去，速率就会降低
* 二是连接间隔， 连接间隔越小，发送数据的速率就越快，可以通过 `esp_ble_gap_update_conn_params` 进行更新(协商)，有下限

其他设置比如调高 UART 波特率、CPU 频率及 Host 和 Controller 运行在双核上(如果有)等优化，可以间接提升吞吐速率

如果周围环境比较嘈杂(电磁干扰，wifi、蓝牙设备比较多) 也会对吞吐量数据产生影响， 可放屏蔽箱里进行测试










# PROGRAMING FULL IMAGE (BIM + APP IMAGE)
1. Xóa hết bộ nhớ, spi flash
sử dụng file `E:\ti\simplelink_cc13x2_26x2_sdk_4_40_00_44\examples\rtos\CC1352P_2_LAUNCHXL\easylink\hexfiles\offChipOad\erase_storage_offchip_cc13x2lp.hex`
2. nạp bim chip (dùng phần mềm `smartRF Flash Programmer 2`, chọn erase)

3. nạp file oad.bin(dùng phần mềm `smartRF Flash Programmer 2`, bỏ chọn erase)

# BOOTLOADER SEQUENCE 
## trong project bim_offship, function Bim_checkImages
1. Nháy led 20 lần, mỗi lần cách nhau 50ms
2. Kiểm tra có image ở chip nhớ ngoài hay không ?
    + Nếu có thì copy vào bộ nhớ chính xong boot
    + Nếu không thì chuyển qua bước 3
3. Kiểm tra có image ở chip nội hay không ?
    + Nếu có thì boot
    + nếu không thì chuyển qua bước 4
4. Kiểm tra xem có factory image không
    + Nếu có thì copy vào bộ nhớ chính rồi boot
    + Nếu không thì loop, nháy led 100ms 1 lần

# APPLICATION SEQUENCE AFTER BOOT
1. chạy ứng dụng
2. Nếu define cờ `FACTORY_IMAGE` và `EXTERNAL_IMAGE_CHECK` trong ota_client.h
    + kiểm tra xem có bộ nhớ ngoài hay không
    + Kiểm tra xem bộ nhớ ngoài có factory image chưa 
    + nếu không có thì nạp phần mềm đang chạy vào phân vùng factory image
```
static void Bim_checkImages(void)
{
    /* Find executable on offchip flash; Validate it before copying to internal flash
     * and then execute it */
    powerUpGpio();
    blinkLed(GREEN_LED, 20, 50);
    powerDownGpio();
    // Check image on external flash, not check factory
    checkImagesExtFlash();

    /* In no valid image has been found in external flash, then
     * try to find a valid executable image on on-chip flash and execute */
    checkImagesIntFlash(0);

    /* BIM is not able find any valid application, either on off-chip
     * or on-chip flash. Try to revert to Factory image */
    Bim_revertFactoryImage();
    while(1)
    {
        powerUpGpio();
        blinkLed(GREEN_LED, 20, 100);
        powerDownGpio();
    }
}
```

# Cách cập nhật file mới vào OTA server
## Tạo firmware
- Trong `Project->properties->Build->Steps->Post-build-steps`
- Sửa phần `BEBE 2652 00000002` thành version phù hợp
```
${COM_TI_SIMPLELINK_CC13X2_26X2_SDK_INSTALL_DIR}/tools/zstack/zigbee_ota_image_converter/zOTAfileGen ${PROJECT_LOC}/${ConfigName}/${ProjName}_oad.bin ${PROJECT_LOC}/${ConfigName}/ BEBE 2652 00000002
```

- Trong `Project->properties->Build->Arm Compiler->Predefined Symbols`
```
OTA_MANUFACTURER_ID=0xBEBE
OTA_TYPE_ID=0x2652
OTA_APP_VERSION=0x00000002
```
## Tạo factory image

To create a Zigbee Factory Image to store in the external flash, define `FACTORY_IMAGE`` and EXTERNAL_IMAGE_CHECK` inside an `OTA client project`

## Đưa firmware mới lên OTA server
1. Trên terminal, run:
```
Git clone https://github.com/lampl92/zigbee-OTA.git
```
2. Copy .zigbee file tạo ở bước 1 vào thư mục zigbee-OTA/images/Ees/
3. Trên terminal, run:
```
Run node scripts/add.js BEBE-2652-00000002.zigbee
```
4. vào file index.json để kiểm tra
## zigbee-OTA
A collection of Zigbee OTA files, see `index.json` for an overview of all available firmware files.

### Adding new and updating existing OTA files
1. Go to this directory
2. Execute `node scripts/add.js PATH_TO_OTA_FILE_OR_URL`, e.g.:
    - `node scripts/add.js ~/Downloads/WhiteLamp-Atmel-Target_0105_5.130.1.30000_0012.sbl-ota`
    - `node scripts/add.js http://fds.dc1.philips.com/firmware/ZGB_100B_010D/1107323831/Sensor-ATmega_6.1.1.27575_0012.sbl-ota`
3. Create a PR

### Updating all existing OTA entries (if add.js has been changed)
1. Go to this directory
2. Execute `node scripts/updateall.js`
3. Create a PR
### Modify Zigbee2mqtt
- Edit `zigbee2mqtt\node_modules\zigbee-herdsman-converters\lib\ota\zigbeeOTA.js`:
```
const url = 'https://raw.githubusercontent.com/lampl92/zigbee-OTA/master/index.json';
```

## Troubleshot
```
Update of '0x00124b001ca1b9df' failed (Cannot read property 'ID' of undefined)
```
- Solution: Tắt zigbee2mqtt, mở lại

# OTA updates
*An ongoing discussion about this feature can be found here: [#2921](https://github.com/Koenkk/zigbee2mqtt/issues/2921)*

This feature allows to update your Zigbee devices over-the-air.

Not all manufacturers make their updates available, therefore only the following devices support it:
- IKEA TRÅDFRI devices
- Ubysis devices
- Some Xiaomi devices
- Salus SP600 Smart plug
- Osram/Ledvance devices (not every firmware is made available by them, in case not you will see the following exception in the log `No image available for ...`)
- Philips Hue devices (not every firmware is made available by them, in case not you will see the following exception in the log `No image available for ...`)
- Jung ZLLxx5004M, Jung ZLLHS4 and Gira 2435-10
Gira does unfortunately not seem to offer firmware updates for their wall transmitter 2430-100 (which is very similar to the Jung ZLLxx5004M) and the update file for the Jung wall transmitter does not work for Gira (probably because the Gira wall transmitter only has 6 buttons instead of 8 on the Jung).
- Sengled devices
- Some Xiaomi devices


## Automatic checking for available updates
Your zigbee devices can request a firmware update check. Zigbee2MQTT obliges this, and will automatically check if updates are available for your devices.

The update state will be published to `zigbee2mqtt/[DEVICE_FRIENLDY_NAME]`, example payload: `{"update": {"state": "available"}}`.
The possible states are:
- `available`: an update is available for this device
- `updating`: update is in progress. During this the progress in % and remaining time in seconds is also added to the payload, example: `{"update": {"state": "updating","progress":13.37,"remaining": 219}}`.
- `idle`: no update available/in progress

To protect privacy it is possible to limit how often third party servers may be contacted. You can set the minimum time that should pass between two firmware update checks, in minutes. The default is 1440 minutes (1 day). Here it is set to check at most every two days:

```yaml
ota:
    update_check_interval: 2880
```

It is also possible to completely ignore these device-initiated requests for updates checks by modifying the configuration.yaml file. In the example below, only manual firmware update checks will be possible:

```yaml
ota:
    disable_automatic_update_check: true
```


*NOTE: there is also a property `update_available` which is deprecated*.

## Manually check if an update is available
To check if an update is available for your device send a message to `zigbee2mqtt/bridge/request/device/ota_update/check` with payload `{"id": "deviceID"}` or `deviceID` where deviceID can be the `ieee_address` or `friendly_name` of the device. Example; request: `{"id": "my_remote"}` or `my_remote`, response: `{"data":{"id": "my_remote","updateAvailable":true},"status":"ok"}`.

## Update to latest firmware
Once an update is available you can update it by sending to `zigbee2mqtt/bridge/request/device/ota_update/update` with payload `{"id": "deviceID"}` or `deviceID` where deviceID can be the `ieee_address` or `friendly_name` of the device, example request: `{"id": "my_remote"}` or `my_remote`. Once the update is completed a response is send, example response: `{"data":{"id": "my_remote","from":{"software_build_id":1,"date_code":"20190101"},"to":{"software_build_id":2,"date_code":"20190102"}},"status":"ok"}`.

An update typically takes +- 10 minutes. While a device is updating a lot of traffic is generated on the network, therefore it is not recommend to execute multiple updates at the same time.


## Troubleshooting
- `Device didn't respond to OTA request` or `Update failed with reason: 'aborted by device'`: try restarting the device by disconnecting the power/battery for a few seconds and try again.
- For battery powered devices make sure that the battery is 70%+ as OTA updating is very power consuming.
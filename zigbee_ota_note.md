# Nạp phần mềm để boot
1. Xóa hết bộ nhớ, spi flash
sử dụng file E:\ti\simplelink_cc13x2_26x2_sdk_4_40_00_44\examples\rtos\CC1352P_2_LAUNCHXL\easylink\hexfiles\offChipOad\erase_storage_offchip_cc13x2lp.hex
2. nạp bim chip (dùng phần mềm smartRF Flash Programmer 2, chọn erase)

3. nạp file oad.bin(dùng phần mềm smartRF Flash Programmer 2, bỏ chọn erase)

4. to create a Zigbee Factory Image to store in the external flash, define FACTORY_IMAGE and EXTERNAL_IMAGE_CHECK inside an OTA client project

# Chu trình bootloader 
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

# Chu trình application sau khi boot
1. chạy phần mềm bt 
2. Nếu define cờ FACTORY_IMAGE và EXTERNAL_IMAGE_CHECK trong ota_client.h
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
Trong Project->properties->Build->Steps->Post-build-steps
```
${COM_TI_SIMPLELINK_CC13X2_26X2_SDK_INSTALL_DIR}/tools/zstack/zigbee_ota_image_converter/zOTAfileGen ${PROJECT_LOC}/${ConfigName}/${ProjName}_oad.bin ${PROJECT_LOC}/${ConfigName}/ BEBE 2652 00000002
```
Sửa phần BEBE 2652 00000002 thành version phù hợp

Trong Project->properties->Build->Arm Compiler->Predefined Symbols
```
OTA_MANUFACTURER_ID=0xBEBE
OTA_TYPE_ID=0x2652
OTA_APP_VERSION=0x00000002
```
#include "RMaker.h"
#include "WiFi.h"
#include "WiFiProv.h"
#include <OneButton.h>
#include <EEPROM.h>

#define DEFAULT_POWER_MODE true
#define DEFAULT_DIMMER_LEVEL 50
const char *service_name = "PROV_1234";
const char *pop = "abcd1234";

static int gpio_button1 = 13;
static int gpio_dimmer1 = 15;

bool dimmer_state = true;

static Device *my_device = NULL;

OneButton button(gpio_button1, false, false);  // Active high, normally pulled down
int brightness = 0;          // Initial brightness
int eepromAddress = 0;       // EEPROM address to store brightness level

unsigned long previousMillis = 0;
const long interval = 100;  // Set your desired interval in milliseconds

void sysProvEvent(arduino_event_t *sys_event)
{
    switch (sys_event->event_id) {
    case ARDUINO_EVENT_PROV_START:
#if CONFIG_IDF_TARGET_ESP32S2
        Serial.printf("\nProvisioning Started with name \"%s\" and PoP \"%s\" on SoftAP\n", service_name, pop);
        printQR(service_name, pop, "softap");
#else
        Serial.printf("\nProvisioning Started with name \"%s\" and PoP \"%s\" on BLE\n", service_name, pop);
        printQR(service_name, pop, "ble");
#endif
        break;
    case ARDUINO_EVENT_PROV_INIT:
        wifi_prov_mgr_disable_auto_stop(10000);
        break;
    case ARDUINO_EVENT_PROV_CRED_SUCCESS:
        wifi_prov_mgr_stop_provisioning();
        break;
    default:;
    }
}

void write_callback(Device *device, Param *param, const param_val_t val, void *priv_data, write_ctx_t *ctx)
{
    const char *device_name = device->getDeviceName();
    const char *param_name = param->getParamName();

    if (strcmp(param_name, "Power") == 0) {
        Serial.printf("Received value = %s for %s - %s\n", val.val.b ? "true" : "false", device_name, param_name);
        if (val.val.b == false) {
            dimmer_state = false;
            brightness = 0;  // Set brightness to 0
            analogWrite(gpio_dimmer1, LOW);
        } else {
            dimmer_state = true;
            brightness = 102;  // Set brightness to 40%
            analogWrite(gpio_dimmer1, brightness);
        }
        // Save power state and brightness level to EEPROM
        EEPROM.write(eepromAddress, dimmer_state);
        EEPROM.write(eepromAddress + 1, brightness);
        EEPROM.commit();

        // Update RainMaker parameters
        my_device->updateAndReportParam(ESP_RMAKER_DEF_POWER_NAME, (bool)dimmer_state);
        my_device->updateAndReportParam("Level", (int)map(brightness, 0, 255, 0, 100));

        param->updateAndReport(val);
    } else if (strcmp(param_name, "Level") == 0) {
        Serial.printf("\nReceived value = %d for %s - %s\n", val.val.i, device_name, param_name);
        brightness = map(val.val.i, 0, 100, 0, 255); // Map the brightness from 0-100 to 0-255
        analogWrite(gpio_dimmer1, brightness);
        param->updateAndReport(val);

        // Update power state and save to EEPROM
        dimmer_state = (brightness > 0);
        EEPROM.write(eepromAddress, dimmer_state);

        // Save brightness level to EEPROM
        EEPROM.write(eepromAddress + 1, brightness);
        EEPROM.commit();

        // Check if the brightness level is 0 and update the power state accordingly
        if (brightness == 0) {
            dimmer_state = false;
            my_device->updateAndReportParam(ESP_RMAKER_DEF_POWER_NAME, (bool)dimmer_state);
        } else {
            my_device->updateAndReportParam(ESP_RMAKER_DEF_POWER_NAME, (bool)dimmer_state);
            my_device->updateAndReportParam("Level", (int)map(brightness, 0, 255, 0, 100));
        }
    }
}

void setup()
{
    Serial.begin(115200);
    pinMode(gpio_button1, INPUT);
    pinMode(gpio_dimmer1, OUTPUT);
    digitalWrite(gpio_dimmer1, DEFAULT_POWER_MODE);

    // Initialize EEPROM with 512 bytes of memory
    EEPROM.begin(512);

    Node my_node;
    my_node = RMaker.initNode("ESP RainMaker Node");
    my_device = new Device("Dimmer", "custom.device.dimmer", &gpio_dimmer1);
    if (!my_device) {
        return;
    }
    my_device->addNameParam();
    my_device->addPowerParam(DEFAULT_POWER_MODE);
    my_device->assignPrimaryParam(my_device->getParamByName(ESP_RMAKER_DEF_POWER_NAME));

    Param level_param("Level", "custom.param.level", value(DEFAULT_DIMMER_LEVEL), PROP_FLAG_READ | PROP_FLAG_WRITE);
    level_param.addBounds(value(0), value(100), value(1));
    level_param.addUIType(ESP_RMAKER_UI_SLIDER);
    my_device->addParam(level_param);

    my_device->addCb(write_callback);
 
    my_node.addDevice(*my_device);

    RMaker.enableOTA(OTA_USING_TOPICS);

    RMaker.enableTZService();

    RMaker.enableSchedule();

    RMaker.enableScenes();

    RMaker.start();

    WiFi.onEvent(sysProvEvent);
#if CONFIG_IDF_TARGET_ESP32S2
    WiFiProv.beginProvision(WIFI_PROV_SCHEME_SOFTAP, WIFI_PROV_SCHEME_HANDLER_NONE, WIFI_PROV_SECURITY_1, pop, service_name);
#else
    WiFiProv.beginProvision(WIFI_PROV_SCHEME_BLE, WIFI_PROV_SCHEME_HANDLER_FREE_BTDM, WIFI_PROV_SECURITY_1, pop, service_name);
#endif

    // Load power state and brightness level from EEPROM
    dimmer_state = EEPROM.read(eepromAddress);
    brightness = EEPROM.read(eepromAddress + 1);
    analogWrite(gpio_dimmer1, brightness);

    button.attachClick(singlePress);
    button.attachDuringLongPress(increaseBrightness);
    button.attachDoubleClick(fullBrightness);
}

void loop()
{
    button.tick();
}

void singlePress() {
    // Toggle LED on and off with 40% brightness
    if (brightness == 0) {
        brightness = 102;  // 40% of 255
        dimmer_state = true;  // LED is on
    } else {
        brightness = 0;
        dimmer_state = false;  // LED is off
    }
    analogWrite(gpio_dimmer1, brightness);

    // Save power state and brightness level to EEPROM
    EEPROM.write(eepromAddress, dimmer_state);
    EEPROM.write(eepromAddress + 1, brightness);
    EEPROM.commit();

    int brightnessPercentage = (brightness * 100) / 255;
    Serial.println("Single Press, Brightness: " + String(brightnessPercentage) + "%");

    // Update RainMaker parameters
    my_device->updateAndReportParam(ESP_RMAKER_DEF_POWER_NAME, (bool)dimmer_state);
    my_device->updateAndReportParam("Level", (int)map(brightness, 0, 255, 0, 100));
}

void increaseBrightness() {
    // Increase brightness linearly on long press
    unsigned long currentMillis = millis();

    if (currentMillis - previousMillis >= interval) {
        brightness += 5;  // Adjust the increment based on your preference
        if (brightness > 255) {
            brightness = 0;  // Wrap around to 0 when it reaches 255
        }
        analogWrite(gpio_dimmer1, brightness);

        // Save power state and brightness level to EEPROM
        dimmer_state = (brightness > 0);
        EEPROM.write(eepromAddress, dimmer_state);
        EEPROM.write(eepromAddress + 1, brightness);
        EEPROM.commit();

        int brightnessPercentage = (brightness * 100) / 255;
        Serial.println("Long Press (Increase Brightness), Brightness: " + String(brightnessPercentage) + "%");

        // Update RainMaker parameters
        my_device->updateAndReportParam(ESP_RMAKER_DEF_POWER_NAME, (bool)dimmer_state);
        my_device->updateAndReportParam("Level", (int)map(brightness, 0, 255, 0, 100));

        // Reset the timer
        previousMillis = currentMillis;
    }
}

void fullBrightness() {
    // Set full brightness on double press
    brightness = 255;
    analogWrite(gpio_dimmer1, brightness);

    // Save power state and brightness level to EEPROM
    dimmer_state = (brightness > 0);
    EEPROM.write(eepromAddress, dimmer_state);
    EEPROM.write(eepromAddress + 1, brightness);
    EEPROM.commit();

    int brightnessPercentage = (brightness * 100) / 255;
    Serial.println("Double Press (Full Brightness), Brightness: " + String(brightnessPercentage) + "%");

    // Update RainMaker parameters
    my_device->updateAndReportParam(ESP_RMAKER_DEF_POWER_NAME, (bool)dimmer_state);
    my_device->updateAndReportParam("Level", (int)map(brightness, 0, 255, 0, 100));
}

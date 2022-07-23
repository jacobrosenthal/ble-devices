# xiaomei WP6003

The basics

* [WP6003 Bluetooth APP Air Quality Detector for PM Formaldehyde TVOC](http://www.vson.com.cn/English/Product/3614894931.html)
* advertises 6003#06003039435F2
* Mac of this device is 60:03:03:94:35:F2
* device name 6003#06003039435F2

Services and characteristics

* 0x1800
  * 0x2a00 Device name
  * 0x2a01 Appearance
  * 0x2a02 Peripheral Privacy Flag
  * 0x2a04 Peripheral Preferred Connection Parameters
* 0x1801
  * 0x2a05 Service Changed
* 0xfff0 Primary Service
  * 0xfff1 write no response
  * 0xfff4 notify,read

## nrf connect

notification received

0A1601080C0C00D45802040300A3010002FF

## APK

```shell
$ adb shell pm list packages | grep smart
package:com.vson.smarthome
$ adb shell pm path com.vson.smarthome
package:/data/app/~~Xa4Wf_N0ZCd2TJ1mm09ZNw==/com.vson.smarthome-lUxL9qtkdaqGICz6rLWZkg==/base.apk
$ adb pull /data/app/~~Xa4Wf_N0ZCd2TJ1mm09ZNw==/com.vson.smarthome-lUxL9qtkdaqGICz6rLWZkg==/base.apk ./smarthome.apk
```

* Install [Vscode apklab](https://braincoke.fr/blog/2021/03/android-reverse-engineering-for-beginners-decompiling-and-patching/#patching-with-vscode-and-apklab)
* "Open Apk" select smarthome.apk and select "decompile java"
* we know our source is at java_src/com/vson/smarthome search within for write (blewrite of some kind) `sendDataToDevice` looks interesting

```java
    public boolean WriteCorrectAltitudeSetting(int i) {
        return sendDataToDevice(new byte[]{-83, (byte) ((i >>> 8) & 255), (byte) (i & 255)});
    }
```

## Googling

* [omarghader](https://github.com/omarghader/airqualiry-wp6003-ble)

```golang
temperature = float32(int32(req[6])<<8+int32(req[7])) / 10
tvoc = float32(int32(req[10])<<8+int32(req[11])) / 1000
hcho = float32(int32(req[12])<<8+int32(req[13])) / 1000
co2 = int32(req[16])<<8 + int32(req[17]) - 150
log.Printf("%f %f %f %d\n", temperature, tvoc, hcho, co2)
```

Interesting, he subtracts 15- from co2...

* [saso5 js](https://github.com/saso5/saso5.github.io/blob/master/WP6003-air-box/index.html)
* [saso5 python](https://github.com/saso5/wp6003)

He does not subtract?

```js
let time = new Date().toLocaleString();
let temp  = value.getInt16(6) / 10.0;
let tvoc  = value.getUint16(10) /1000.0;
let hcho  = value.getUint16(12) /1000.0;
let co2   = value.getUint16(16);
```

`Device6003Activity.java`

```java
    /* JADX INFO: Access modifiers changed from: private */
    public List<UploadRecordBean> getRecordList(String[] strArr, String str, String str2, String str3) {
        int parseInt = Integer.parseInt(strArr[10].concat(strArr[11]), 16);
        int parseInt2 = Integer.parseInt(strArr[12].concat(strArr[13]), 16);
        float handleNegativeTemp = Activity6003ViewModel.handleNegativeTemp(Integer.parseInt(strArr[6].concat(strArr[7]), 16));
        if (TextUtils.isEmpty(str2) || TextUtils.isEmpty(str3)) {
            UploadRecordBean[] uploadRecordBeanArr = new UploadRecordBean[4];
            uploadRecordBeanArr[0] = new UploadRecordBean(ExifInterface.GPS_MEASUREMENT_2D, this.mBtAddress, String.valueOf(handleNegativeTemp), str, this);
            uploadRecordBeanArr[1] = new UploadRecordBean("4", this.mBtAddress, String.valueOf((parseInt == 16383 ? 0 : parseInt) / 1000.0f), str, this);
            String str4 = this.mBtAddress;
            if (parseInt2 == 16383) {
                parseInt2 = 0;
            }
            uploadRecordBeanArr[2] = new UploadRecordBean("5", str4, String.valueOf(parseInt2 / 1000.0f), str, this);
            String str5 = this.mBtAddress;
            if (parseInt == 16383) {
                parseInt = 0;
            }
            uploadRecordBeanArr[3] = new UploadRecordBean("6", str5, Activity6003ViewModel.handleCo2RealtimeData(parseInt / 1000.0f, Integer.parseInt(strArr[16].concat(strArr[17]), 16)), str, this);
            return Arrays.asList(uploadRecordBeanArr);
        }
        UploadRecordBean[] uploadRecordBeanArr2 = new UploadRecordBean[4];
        uploadRecordBeanArr2[0] = new UploadRecordBean(ExifInterface.GPS_MEASUREMENT_2D, this.mBtAddress, String.valueOf(handleNegativeTemp), str, str2, str3);
        uploadRecordBeanArr2[1] = new UploadRecordBean("4", this.mBtAddress, String.valueOf((parseInt == 16383 ? 0 : parseInt) / 1000.0f), str, str2, str3);
        String str6 = this.mBtAddress;
        if (parseInt2 == 16383) {
            parseInt2 = 0;
        }
        uploadRecordBeanArr2[2] = new UploadRecordBean("5", str6, String.valueOf(parseInt2 / 1000.0f), str, str2, str3);
        String str7 = this.mBtAddress;
        if (parseInt == 16383) {
            parseInt = 0;
        }
        uploadRecordBeanArr2[3] = new UploadRecordBean("6", str7, Activity6003ViewModel.handleCo2RealtimeData(parseInt / 1000.0f, Integer.parseInt(strArr[16].concat(strArr[17]), 16)), str, str2, str3);
        return Arrays.asList(uploadRecordBeanArr2);
    }
```

 6  7 temperature
10 11 parseint tvoc (from omargader)
12 13 parseint2 hcoc (from omargader)
16 17 c02

using tvoc to modify co2? `handleCo2RealtimeData`

`Activity6003ViewModel.java`

```java
public static String handleCo2RealtimeData(float f2, int i) {
    int i2;
    Random random = new Random();
    if (f2 <= 1.0f) {
        i2 = Math.round(((i - 50) * 0.85f) + random.nextInt(10));
    } else {
        i2 = Math.round((i - 50) + random.nextInt(10) + 10);
    }
    if (i2 < 400) {
        i2 = 400;
    } else if (i2 > 2000) {
        i2 = 2000;
    }
    return String.valueOf(i2);
}
```

cleaned up version with comments

```java
    public static String handleCo2RealtimeData(float tvoc, int rawco2) {
        int i2;
        Random random = new Random();
        // if tvoc is low, (rawco2-50) *.85 and jitter???
        if (tvoc <= 1.0f) {
            i2 = Math.round(((rawco2 - 50) * 0.85f) + random.nextInt(10));
        }// if high but we dont lower it 
        else {
            i2 = Math.round((rawco2 - 50) + random.nextInt(10) + 10);
        }
        // clamp between 
        if (i2 < 400) {
            i2 = 400;
        } else if (i2 > 2000) {
            i2 = 2000;
        }
        return String.valueOf(i2);
    }
```

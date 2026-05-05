# Lab — `vhal-prop` (Day 67 / Day 73 capstone)

Add a custom OEM Vehicle property to the AIDL VHAL on `aosp_cf_x86_64_auto-userdebug` and read it from a Car app.

## Files
```
vhal-prop/
├── README.md
├── vhal/
│   ├── Android.bp
│   ├── DefaultVehicleHal.cpp.diff   ← patch into reference VHAL
│   └── PropertyConfig.json           ← config snippet
└── carapp/
    ├── Android.bp
    └── src/.../MainActivity.kt
```

## 1. Choose a property ID

Use a `VENDOR` group ID (top byte = 0x21000000) with a unique low 16 bits. Example:
```
CUSTOM_OEM_PROP = 0x21400500
              = VehiclePropertyGroup.VENDOR
              | VehiclePropertyType.INT32
              | VehicleArea.GLOBAL
              | 0x0500
```

## 2. Patch the reference VHAL

`device/generic/car/emulator/vhal_aidl/VehicleEmulator/DefaultProperties.json` (or equivalent for your tree) — add:
```json
{
  "property": "CUSTOM_OEM_PROP",
  "propId": 559939840,
  "type": "INT32",
  "access": "READ_WRITE",
  "changeMode": "ON_CHANGE",
  "defaultValue": { "int32Values": [0] }
}
```

Rebuild VHAL, redeploy, and:
```bash
cf:# dumpsys car_service | grep -A2 0x21400500
cf:# cmd car_service inject-vhal-event 0x21400500 i32 42
```

## 3. Car app

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var car: Car
    private lateinit var prop: CarPropertyManager
    private val ID = 0x21400500

    override fun onCreate(s: Bundle?) {
        super.onCreate(s); setContentView(R.layout.main)
        car = Car.createCar(this) ?: return
        prop = car.getCarManager(Car.PROPERTY_SERVICE) as CarPropertyManager
        findViewById<Button>(R.id.read).setOnClickListener {
            val v = prop.getIntProperty(ID, 0 /* areaId GLOBAL */)
            findViewById<TextView>(R.id.value).text = v.toString()
        }
    }
}
```

`AndroidManifest.xml` needs:
```xml
<uses-permission android:name="android.car.permission.CAR_VENDOR_EXTENSION"/>
```

## Verify
```bash
cf:# am start -n com.example.vhalapp/.MainActivity
cf:# cmd car_service inject-vhal-event 0x21400500 i32 99
# Tap "read" in the app → value 99
```


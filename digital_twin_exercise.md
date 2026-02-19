
# Unity Digital Twin Øvelse

Denne øvelse guider de studerende gennem at bygge en simpel digital twin baseret på binære signaler fra et produktionssystem. Øvelsen tager udgangspunkt i Unity 3D og C# og kan gennemføres på cirka 4 timer.

## Læringsmål
- Lære at indlæse og afspille tidsseriedata (CSV) i Unity.
- Forstå digital twins i en forsimplet, visuel form.
- Opbygge en modulær arkitektur: DataLoader → SignalPlayer → Controllers.
- Visualisere tilstandsdata i 3D med simple indikatorer og objekter.

## Projektstruktur
```
UnityProject/
  Assets/
    Scenes/
      VpsTwin.unity
    Scripts/
      Data/
        CsvLoader.cs
        VpsSample.cs
      Playback/
        SignalPlayer.cs
      Modules/
        DeliveryController.cs
        RotaryTableController.cs
        Indicator.cs
    StreamingAssets/
      vps_demo.csv
    Prefabs/
      DeliveryModule.prefab
      RotaryTable.prefab
      Bottle.prefab
      IndicatorLamp.prefab
```

## Eksempel-CSV (vps_demo.csv)
```csv
timestamp_ms,conveyor_on,aspirator_on,slot2,slot3,slot5,slot6,slot8
0,0,0,0,0,0,0,0
100,1,1,0,0,0,0,0
200,1,1,1,0,0,0,0
300,1,1,1,1,0,0,0
400,1,1,1,1,1,0,0
500,1,1,1,1,1,1,0
600,1,1,0,1,1,1,1
700,0,0,0,0,0,0,1
800,0,0,0,0,0,0,0
```

## Kodefiler

### VpsSample.cs
```csharp
using System;

[Serializable]
public class VpsSample
{
    public long timestampMs;
    public int conveyor_on;
    public int aspirator_on;
    public int slot2;
    public int slot3;
    public int slot5;
    public int slot6;
    public int slot8;
}
```

### CsvLoader.cs
```csharp
using System.Collections.Generic;
using System.Globalization;
using System.IO;
using UnityEngine;

public static class CsvLoader
{
    public static List<VpsSample> Load(string fileNameInStreamingAssets)
    {
        var path = Path.Combine(Application.streamingAssetsPath, fileNameInStreamingAssets);
        var lines = File.ReadAllLines(path);
        var list = new List<VpsSample>();
        if (lines.Length <= 1) return list;

        for (int i = 1; i < lines.Length; i++)
        {
            var line = lines[i].Trim();
            if (string.IsNullOrWhiteSpace(line)) continue;
            var cols = line.Split(',');

            var s = new VpsSample
            {
                timestampMs   = long.Parse(cols[0], CultureInfo.InvariantCulture),
                conveyor_on   = int.Parse(cols[1]),
                aspirator_on  = int.Parse(cols[2]),
                slot2         = int.Parse(cols[3]),
                slot3         = int.Parse(cols[4]),
                slot5         = int.Parse(cols[5]),
                slot6         = int.Parse(cols[6]),
                slot8         = int.Parse(cols[7])
            };
            list.Add(s);
        }
        return list;
    }
}
```

### Indicator.cs
```csharp
using UnityEngine;

[RequireComponent(typeof(MeshRenderer))]
public class Indicator : MonoBehaviour
{
    public Color offColor = Color.gray;
    public Color onColor  = Color.green;

    private MeshRenderer _mr;

    void Awake()
    {
        _mr = GetComponent<MeshRenderer>();
        Set(false);
    }

    public void Set(bool on)
    {
        _mr.material.color = on ? onColor : offColor;
    }
}
```

### DeliveryController.cs
```csharp
using UnityEngine;

public class DeliveryController : MonoBehaviour
{
    public Indicator conveyorIndicator;
    public Indicator aspiratorIndicator;

    public void Apply(VpsSample s)
    {
        conveyorIndicator?.Set(s.conveyor_on == 1);
        aspiratorIndicator?.Set(s.aspirator_on == 1);
    }
}
```

### RotaryTableController.cs
```csharp
using UnityEngine;

public class RotaryTableController : MonoBehaviour
{
    public Indicator slot2Ind;
    public Indicator slot3Ind;
    public Indicator slot5Ind;
    public Indicator slot6Ind;
    public Indicator slot8Ind;

    public GameObject slot2Bottle;
    public GameObject slot3Bottle;
    public GameObject slot5Bottle;
    public GameObject slot6Bottle;
    public GameObject slot8Bottle;

    public void Apply(VpsSample s)
    {
        SetSlot(slot2Ind, slot2Bottle, s.slot2 == 1);
        SetSlot(slot3Ind, slot3Bottle, s.slot3 == 1);
        SetSlot(slot5Ind, slot5Bottle, s.slot5 == 1);
        SetSlot(slot6Ind, slot6Bottle, s.slot6 == 1);
        SetSlot(slot8Ind, slot8Bottle, s.slot8 == 1);
    }

    private void SetSlot(Indicator ind, GameObject bottle, bool present)
    {
        ind?.Set(present);
        if (bottle != null) bottle.SetActive(present);
    }
}
```

### SignalPlayer.cs
```csharp
using System.Collections.Generic;
using UnityEngine;

public class SignalPlayer : MonoBehaviour
{
    public string csvFileName = "vps_demo.csv";
    public float timeScale = 1.0f;

    public DeliveryController delivery;
    public RotaryTableController rotary;

    private List<VpsSample> _samples;
    private int _index;
    private long _t0;
    private float _elapsed;

    void Start()
    {
        _samples = CsvLoader.Load(csvFileName);
        if (_samples.Count == 0) { enabled = false; return; }

        _t0 = _samples[0].timestampMs;
        _index = 0;
        Apply(_samples[0]);
    }

    void Update()
    {
        if (_index >= _samples.Count - 1) return;

        _elapsed += Time.deltaTime * Mathf.Max(0.0001f, timeScale);
        var simTimeMs = _t0 + (long)(_elapsed * 1000f);

        while (_index + 1 < _samples.Count && _samples[_index + 1].timestampMs <= simTimeMs)
        {
            _index++;
            Apply(_samples[_index]);
        }
    }

    private void Apply(VpsSample s)
    {
        delivery?.Apply(s);
        rotary?.Apply(s);
    }
}
```

## Afslutning
Når alle scripts er tilknyttet deres objekter i Unity, og `vps_demo.csv` ligger i *StreamingAssets*, kan scenen afspilles. Indikatorer og flasker skifter nu tilstand i takt med data.

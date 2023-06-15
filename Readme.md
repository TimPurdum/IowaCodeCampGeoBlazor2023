# Using Maps in Blazor
## Iowa Code Camp 2023
Tim Purdum

tim.purdum@dymaptic.com

<a href="https://fosstodon.org/@TimPurdum">@TimPurdum@fosstodon.org</a>

https://timpurdum.dev

<br/>

## Setting Up

[Getting Started | GeoBlazor](https://docs.geoblazor.com/pages/gettingStarted.html)

```bash
dotnet new blazorserver-empty
dotnet add package dymaptic.GeoBlazor.Core
```

[API keys | ArcGIS Developers](https://developers.arcgis.com/api-keys/)

### `appsettings.json` or `secrets.json`

```json
"ArcGISApiKey": "YOUR KEY HERE"
```

### `_Host.cshtml`

```html
<link href="_content/dymaptic.GeoBlazor.Core"/>
<link href="_content/dymaptic.GeoBlazor.Core/assets/esri/themes/dark/main.css" rel="stylesheet" />
<link href="ICC-GB.styles.css" rel="stylesheet" />
```

### `_Imports.razor`

```html
@using dymaptic.GeoBlazor.Core.Components
@using dymaptic.GeoBlazor.Core.Components.Geometries
@using dymaptic.GeoBlazor.Core.Components.Layers
@using dymaptic.GeoBlazor.Core.Components.Popups
@using dymaptic.GeoBlazor.Core.Components.Symbols
@using dymaptic.GeoBlazor.Core.Components.Views
@using dymaptic.GeoBlazor.Core.Components.Widgets
@using dymaptic.GeoBlazor.Core.Objects
```

### `Program.cs`

```csharp
using dymaptic.GeoBlazor.Core;
using Microsoft.AspNetCore.StaticFiles;
...
builder.Services.AddGeoBlazor();
...
var provider = new FileExtensionContentTypeProvider();
provider.Mappings[".wsv"] = "application/octet-stream";

app.UseStaticFiles();
// NOTE: for some reason, you still need the plain "UseStaticFiles" call above
app.UseStaticFiles(new StaticFileOptions
{
    ContentTypeProvider = provider
});
```

[Map | API Reference | ArcGIS Maps SDK for JavaScript 4.26 | ArcGIS Developers](https://developers.arcgis.com/javascript/latest/api-reference/esri-Map.html#basemap)

## Display a Map

### `Index.razor`

```html
<h1>Des Moines Transit</h1>
<MapView Longitude="-93.598022" Latitude="41.619549" Zoom="11" 
         Style="height: 80vh; width: 90vw; margin-left: 5vw;"
         @ref="mapView">
    <Map ArcGISDefaultBasemap="arcgis-streets"></Map>
</MapView>
```

```csharp
private MapView? mapView;
```

## Add DART Routes & Stops

[Routes | Routes | Iowa Department of Transportation - Open Data (arcgis.com)](https://public-iowadot.opendata.arcgis.com/datasets/routes/explore)

```html
<Map ArcGISDefaultBasemap="arcgis-streets">
    <FeatureLayer @ref="routesLayer" Url="https://services.arcgis.com/8lRhdTsQyJpO52F1/arcgis/rest/services/IowaGTFS_View/FeatureServer/0/" />
    <FeatureLayer Url="https://services.arcgis.com/8lRhdTsQyJpO52F1/arcgis/rest/services/IowaGTFS_View/FeatureServer/1" />
</Map>
```

```csharp
private FeatureLayer? routesLayer;
```

## Add Widgets

```html
<LayerListWidget Position="OverlayPosition.TopRight" />
<LocateWidget Position="OverlayPosition.TopLeft" />
<ScaleBarWidget Position="OverlayPosition.BottomLeft" />
```

## Add Popup

```html
<Map ArcGISDefaultBasemap="arcgis-streets">
    <FeatureLayer ...>
        <PopupTemplate @ref="popupTemplate" Title="{trip_headsign}" 
                       StringContent="Bus Route {route_long}">
        </PopupTemplate>
    </FeatureLayer>
    <FeatureLayer ... />
</Map>
<PopupWidget @ref="popupWidget" DockEnabled="true">
    <PopupDockOptions ButtonEnabled="false"
                      BreakPoint="@(new BreakPoint(false))"
                      Position="PopupDockPosition.BottomRight" />
</PopupWidget>
```

```csharp
private PopupTemplate? popupTemplate;
private PopupWidget? popupWidget;
```

## Add Moving Bus Graphic

### Import CSV Data

- Copy `stops` and `stop_times` from [Developer Resources | DART - Des Moines Area Regional Transit Authority (ridedart.com)](https://www.ridedart.com/developer-resources) into `wwwroot`

```bash
dotnet add package CsvHelper
```

### `_Imports.razor`

```csharp
@using CsvHelper
@using CsvHelper.Configuration.Attributes
@using System.Globalization
```

### `Index.razor`

```csharp
[Inject]
IWebHostEnvironment WebHostEnvironment { get; set; } = default!;

protected override void OnInitialized()
{
    GetStopTimeData();
}

private void GetStopTimeData()
{
    if (stopTimeData is not null)
    {
        return;
    }
    stopTimeData = new List<StopTimeData>();
    var filePath = Path.Combine(WebHostEnvironment.WebRootPath, "stop_times.csv");
    using (StreamReader reader = new StreamReader(filePath))
    using (CsvReader csvReader = new CsvReader(reader, CultureInfo.InvariantCulture))
    {
        stopTimeData = csvReader.GetRecords<StopTimeData>().ToList()!;
    }

    filePath = Path.Combine(WebHostEnvironment.WebRootPath, "stops.csv");
    using (StreamReader reader = new StreamReader(filePath))
    using (CsvReader csvReader = new CsvReader(reader, CultureInfo.InvariantCulture))
    {
        stopData = csvReader.GetRecords<StopData>().ToList();
    }
}

private List<StopData>? stopData;
private List<StopTimeData>? stopTimeData;

private record StopTimeData(string trip_id, string arrival_time, string stop_id, int stop_sequence);
private record StopData(string stop_id, string stop_name, double stop_lat, double stop_lon);
```

## Add Click Handler

```csharp
private async Task OnClick(ClickEvent clickEvent)
{
    HitTestOptions options = new()
    {
        IncludeByGeoBlazorId = new[] { routesLayer!.Id }
    };
    HitTestResult result = await mapView!.HitTest(clickEvent, options);
    Graphic? feature = result.Results.OfType<GraphicHit>().FirstOrDefault()?.Graphic;
    if (feature is not null)
    {
        var query = await routesLayer.CreateQuery();
        query.ObjectIds = new[] { (int)(long)feature!.Attributes["OBJECTID"] };
        query.OutFields = new[] { "*" };
        feature = (await routesLayer.QueryFeatures(query))!.Features!.First();
        await DriveTheBus(feature);
    }
}
```

## Add Bus Graphic and Animation

```html
<div style="display: flex; flex-direction: row; justify-content: space-between; margin: 1rem 5vw;">
    <label>
        Bus delay in ms (lower = faster): <input type="number" @bind="busDelay" />
    </label>
</div>
<br />
<br />
```

```csharp
private async Task DriveTheBus(Graphic? feature)
{
    if (cancellationTokenSource is not null)
    {
        cancellationTokenSource.Cancel();
    }

    cancellationTokenSource = new CancellationTokenSource();
    var token = cancellationTokenSource.Token;
    if (feature is null)
    {
        // no route found
        return;
    }

    isPaused = false;
    await mapView!.OpenPopup(new PopupOpenOptions()
    {
        Features = new Graphic[] { feature }
    });

    var stopTimes = stopTimeData!.Where(s => s.trip_id == feature.Attributes["trip_id"].ToString())
        .OrderBy(s => s.stop_sequence).ToList();

    await mapView.AddGraphic(busGraphic!);
    foreach (var stopTime in stopTimes)
    {
        while (isPaused)
        {
            await Task.Delay(100);
        }
        currentStopId = stopTime.stop_id;
        var stop = stopData!.First(s => s.stop_id == stopTime.stop_id);

        var busPoint = new Point(stop.stop_lon, stop.stop_lat);
        await busGraphic!.SetGeometry(busPoint);
        StateHasChanged();
        if (token.IsCancellationRequested)
        {
            return;
        }
        await Task.Delay(busDelay, token);
    }
    await mapView.RemoveGraphic(busGraphic!);
}

private Graphic? busGraphic = new Graphic(new Point(0, 0), new PictureMarkerSymbol("https://static.arcgis.com/images/Symbols/Transportation/Bus.png", 20, 20));
private CancellationTokenSource? cancellationTokenSource = new();
private bool isPaused;
private int busDelay = 1000;
```

## Create Stop Popup Content

```html
<PopupTemplate @ref="popupTemplate" Title="{trip_headsign}" ContentFunction="BuildPopupContent">
```

```csharp
private ValueTask<PopupContent[]> BuildPopupContent(Graphic graphic)
{
    List<PopupContent> popupContents = new();
    if (currentStopId is null)
    {
        return ValueTask.FromResult(popupContents.ToArray());
    }

    var currentStop = stopData!.First(s => s.stop_id == currentStopId);
    var currentStopTime = stopTimeData!.First(s => s.stop_id == currentStopId);
    var stopTime = TimeOnly.Parse(currentStopTime.arrival_time);
    popupContents.Add(new TextPopupContent($"Stop #{currentStopTime.stop_sequence}: {currentStop.stop_name}, {stopTime}"));
    popupContents.Add(new TextPopupContent($"Coordinates: LAT: {currentStop.stop_lat}, LONG: {currentStop.stop_lon}"));

    return ValueTask.FromResult(popupContents.ToArray());
}
```

## Add Pause Action Button

- Place `play.png` and `pause.png` in `wwwroot/images`

```html
<PopupTemplate @ref="popupTemplate" Title="{trip_headsign}" ContentFunction="BuildPopupContent">
    <ActionButton Image="@(isPaused ? "images/play.png" : "images/pause.png")"
                    Title="@(isPaused ? "Play" : "Pause")"
                    Id="play"
                    CallbackFunction="StartOrPause" />
</PopupTemplate>
```

```csharp
private async Task StartOrPause()
{
    await InvokeAsync(() =>
    {
        isPaused = !isPaused;
        StateHasChanged();
    });
}
```

## Add Search

```html
<div style="display: flex; flex-direction: row; justify-content: space-between; margin: 1rem 5vw;">
    <label>
        Find a route: <input type="text" @bind="routeSearchText" />
        <button @onclick="OnSearch">Search</button>
    </label>
    <label>
        Bus delay in ms (lower = faster): <input type="number" @bind="busDelay" />
    </label>
</div>
```

```csharp
private async Task OnSearch()
{
    var query = await routesLayer!.CreateQuery();
    query.OutFields = new string[] { "OBJECTID", "trip_id" };
    query.Where = $"route_shortname LIKE '%{routeSearchText}%' OR route_long LIKE '%{routeSearchText}%' OR trip_headsign LIKE '%{routeSearchText}%'";
    var featureSet = await routesLayer!.QueryFeatures(query);
    if (featureSet?.Features is not null && featureSet.Features.Any())
    {
        Graphic feature = featureSet.Features.First();
        await DriveTheBus(feature);
    }
}

private string routeSearchText = string.Empty;
```


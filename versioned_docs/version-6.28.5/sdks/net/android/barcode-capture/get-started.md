---
sidebar_position: 2
pagination_prev: null
framework: netAndroid
keywords:
  - netAndroid
---

# Get Started

In this guide you will learn step-by-step how to add Barcode Capture to your application.

The general steps are:

- Creating a new Data Capture Context instance
- Create your barcode capture settings and enable the barcode symbologies you want to read
- Create a new barcode capture mode instance and initialize it
- Register a barcode capture listener to receive scan events
- Process successful scans according to your application’s needs and decide whether more codes will be scanned or the scanning process should be stopped
- Obtain a camera instance and set it as the frame source on the data capture context
- Display the camera preview by creating a data capture view
- If displaying a preview, optionally create a new overlay and add it to data capture view for better visual feedback

## Create a Data Capture Context

The first step to add capture capabilities to your application is to create a new [data capture context](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/core/api/data-capture-context.html#class-scandit.datacapture.core.DataCaptureContext). The context expects a valid Scandit Data Capture SDK license key during construction.

```csharp
DataCaptureContext context = DataCaptureContext.ForLicenseKey("-- ENTER YOUR SCANDIT LICENSE KEY HERE --");
```

## Configure the Barcode Scanning Behavior

Barcode scanning is orchestrated by the [BarcodeCapture](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/barcode-capture/api/barcode-capture.html#class-scandit.datacapture.barcode.BarcodeCapture) [data capture mode](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/core/api/data-capture-mode.html#interface-scandit.datacapture.core.IDataCaptureMode). This class is the main entry point for scanning barcodes. It is configured through [BarcodeCaptureSettings](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/barcode-capture/api/barcode-capture-settings.html#class-scandit.datacapture.barcode.BarcodeCaptureSettings) and allows to register one or more [listeners](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/barcode-capture/api/barcode-capture-listener.html#interface-scandit.datacapture.barcode.IBarcodeCaptureListener) that will get informed whenever new codes have been recognized.

For this tutorial, we will setup barcode scanning for a small list of different barcode types, called [symbologies](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/barcode-capture/api/symbology.html#enum-scandit.datacapture.barcode.Symbology). The list of symbologies to enable is highly application specific. We recommend that you only enable the list of symbologies your application requires.

```csharp
BarcodeCaptureSettings settings = BarcodeCaptureSettings.Create();
HashSet<Symbology> symbologies = new HashSet<Symbology>()
{
Symbology.Code128,
Symbology.Code39,
Symbology.Qr,
Symbology.Ean8,
Symbology.Upce,
Symbology.Ean13Upca
};
settings.EnableSymbologies(symbologies);
```

If you are not disabling barcode capture immediately after having scanned the first code, consider setting the [BarcodeCaptureSettings.CodeDuplicateFilter](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/barcode-capture/api/barcode-capture-settings.html#property-scandit.datacapture.barcode.BarcodeCaptureSettings.CodeDuplicateFilter) to around 500 or even \-1 if you do not want codes to be scanned more than once.

Next, create a [BarcodeCapture](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/barcode-capture/api/barcode-capture.html#class-scandit.datacapture.barcode.BarcodeCapture) instance with the settings initialized in the previous step:

```csharp
barcodeCapture = BarcodeCapture.Create(context, settings);
```

## Register the Barcode Capture Listener

To get informed whenever a new code has been recognized, add a [IBarcodeCaptureListener](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/barcode-capture/api/barcode-capture-listener.html#interface-scandit.datacapture.barcode.IBarcodeCaptureListener) through [BarcodeCapture.AddListener()](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/barcode-capture/api/barcode-capture.html#method-scandit.datacapture.barcode.BarcodeCapture.AddListener) and implement the listener methods to suit your application’s needs.

First implement the [IBarcodeCaptureListener](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/barcode-capture/api/barcode-capture-listener.html#interface-scandit.datacapture.barcode.IBarcodeCaptureListener) interface. For example:

```csharp
public void OnBarcodeScanned(BarcodeCapture barcodeCapture, BarcodeCaptureSession session, IFrameData frameData)
{
IList<Barcode> barcodes = session?.NewlyRecognizedBarcode;
// Do something with the barcodes
}
```

Then add the listener:

```csharp
barcodeCapture.AddListener(this);
```

Alternatively to register [IBarcodeCaptureListener](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/barcode-capture/api/barcode-capture-listener.html#interface-scandit.datacapture.barcode.IBarcodeCaptureListener) interface it is possible to subscribe to corresponding events. For example:

```csharp
barcodeCapture.BarcodeScanned += (object sender, BarcodeCaptureEventArgs args) =>
{
IList<Barcode> barcodes = args.Session?.NewlyRecognizedBarcode;
// Do something with the barcodes
}
```

## Use the Built-in Camera

The data capture context supports using different frame sources to perform recognition on. Most applications will use the built-in camera of the device, e.g. the world-facing camera of a device. The remainder of this tutorial will assume that you use the built-in camera.

:::important
In Android, the user must explicitly grant permission for each app to access cameras. Your app needs to declare the use of the Camera permission in the AndroidManifest.xml file and request it at runtime so the user can grant or deny the permission. To do that follow the guidelines from [Permissions in Android](https://learn.microsoft.com/en-us/xamarin/android/app-fundamentals/permissions) to request the android.permission.CAMERA permission.
:::

When using the built-in camera there are recommended settings for each capture mode. These should be used to achieve the best performance and user experience for the respective mode. The following couple of lines show how to get the recommended settings and create the camera from it:

```csharp
camera = Camera.GetDefaultCamera();
camera?.ApplySettingsAsync(BarcodeCapture.RecommendedCameraSettings);
```

Because the frame source is configurable, the data capture context must be told which frame source to use. This is done with a call to [DataCaptureContext.SetFrameSourceAsync()](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/core/api/data-capture-context.html#method-scandit.datacapture.core.DataCaptureContext.SetFrameSourceAsync):

```csharp
context.SetFrameSourceAsync(camera);
```

The camera is off by default and must be turned on. This is done by calling [IFrameSource.SwitchToDesiredStateAsync()](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/core/api/frame-source.html#method-scandit.datacapture.core.IFrameSource.SwitchToDesiredStateAsync) with a value of [FrameSourceState.On](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/core/api/frame-source.html#value-scandit.datacapture.core.FrameSourceState.On):

```csharp
camera?.SwitchToDesiredStateAsync(FrameSourceState.On);
```



## Use a Capture View to Visualize the Scan Process

When using the built-in camera as frame source, you will typically want to display the camera preview on the screen together with UI elements that guide the user through the capturing process. To do that, add a [DataCaptureView](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/core/api/ui/data-capture-view.html#class-scandit.datacapture.core.ui.DataCaptureView) to your view hierarchy:

```csharp
DataCaptureView dataCaptureView = DataCaptureView.Create(this, dataCaptureContext);
SetContentView(dataCaptureView);
```

Alternatively you can use a [DataCaptureView](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/core/api/ui/data-capture-view.html#class-scandit.datacapture.core.ui.DataCaptureView) from XAML in your MAUI application. For example:

```xml
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
xmlns:scandit="clr-namespace:Scandit.DataCapture.Core.UI.Maui;assembly=ScanditCaptureCoreMaui">
<ContentPage.Content>
<AbsoluteLayout>
<scandit:DataCaptureView x:Name="dataCaptureView" AbsoluteLayout.LayoutBounds="0,0,1,1"
AbsoluteLayout.LayoutFlags="All" DataCaptureContext="{Binding DataCaptureContext}">
</scandit:DataCaptureView>
</AbsoluteLayout>
</ContentPage.Content>
</ContentPage>
```

You can configure your view in the code behind class. For example:

```csharp
public partial class MainPage : ContentPage
{
public MainPage()
{
InitializeComponent();

// Initialization of DataCaptureView happens on handler changed event.
dataCaptureView.HandlerChanged += DataCaptureViewHandlerChanged;
}

private void DataCaptureViewHandlerChanged(object? sender, EventArgs e)
{
// Your dataCaptureView configuration goes here, e.g. add overlay
}
}
```

For MAUI development add [Scandit.DataCapture.Core.Maui](https://www.nuget.org/packages/Scandit.DataCapture.Core.Maui) NuGet package into your project.

To visualize the results of barcode scanning, the following [overlay](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/barcode-capture/api/ui/barcode-capture-overlay.html#class-scandit.datacapture.barcode.ui.BarcodeCaptureOverlay) can be added:

```csharp
BarcodeCaptureOverlay overlay = BarcodeCaptureOverlay.Create(barcodeCapture, dataCaptureView);
```

## Disabling Barcode Capture

To disable barcode capture, for instance as a consequence of a barcode being recognized, set [BarcodeCapture.Enabled](https://docs.scandit.com/6.28/data-capture-sdk/dotnet.android/barcode-capture/api/barcode-capture.html#property-scandit.datacapture.barcode.BarcodeCapture.IsEnabled) to _false_.

The effect is immediate: no more frames will be processed _after_ the change. However, if a frame is currently being processed, this frame will be completely processed and deliver any results/callbacks to the registered listeners. Note that disabling the capture mode does not stop the camera, the camera continues to stream frames until it is turned off.

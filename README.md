![Banner](AirlinerAttitudeBanner.png)

# FiremonkeySensorFusion
Accelerometer+Magnetometer+GPS sensor association object. The object calculates rectangular coordinates.
This can be used to power augmented reality apps for mobile devices. 

Cross platform code ( Android and iOS ). Tested w/ Delphi 10.4.1

Tags: #delphi #firemonkey #Sensorfusion #pascal

This project uses DelphiWorlds Android sensor code (KastriFree lib)  by Dave Nottage ( files with DW. prefix )

This component is currently used in apps "CamSextant"  and "PlanetFun"

Works as follows:
1. Get GPS position, to calculate Magnetic Declination. Android offers a WMM service for that. iOS seems to have it too, but I worked the magnetic declination from TrueHeading/MagHeading properties of the GPS device. 
2. Get Accelerometer and Magnetometer 3D vectors
3. Calculate tilt compensated rectangular coordinates* by rotating the magnetometer vector with the accelerometer vector. This results in the magnetic vector in relation to the phone attitude.
4. Apply magnetic declination to obtain True Heading (azimuth)

* ( many names for rectangular coordinates: azimuth/altitude/roll or heading/elevation/roll or pitch/bank/roll )

iOS version uses a 100ms timer to get sensor readings. It would be better to use sensor change events, but I don't know how to do that.

Android version uses native sensor code ( from DelphiWorlds )

Note that iOS GPS sensor has a TrueHeading property, which could be used directly, avoiding all this. But it has a problem when the altitude crosses the 45 degree boundary. The GPS TrueHeading jumps several degrees at that point. My guess is that iOS changes the rectangular coordinates axis when the altitude is more than 45 degrees, which I think is wrong. Not sure.

note: versions before D10.3.3 used a hack to pass int64 via JNI on Android (Delphi JNI had some endian problem  ).
This was commented when the compiler was corrected. Watch if using previous Delphi versions ( i.e. uncomment the hack )
On D10.4.1 it seems the bug is back, so the hack was reintroduced :|

## Usage
* Add unit to uses:  MagnetometerAccelerometerFusion
* Add to the form:  fMagAccelFusion:TMagnetoAccelerometerFusion;
* On FormCreate:

      fMagAccelFusion := TMagnetoAccelerometerFusion.Create(Self);
      // fMagAccelFusion.OnAccelerometerChange  := FusionSensorAccelChanged;          //optional sensor events
      // fMagAccelFusion.OnMagnetometerChange   := FusionSensorMagChanged;
      fMagAccelFusion.OnHeadingAltitudeChange:= FusionSensorHeadingAltitudeChanged;   // combined sensor change handler
    
* Implement sensor handler:  

      procedure TfrmMain.FusionSensorHeadingAltitudeChanged(Sender:TObject);
      begin
        labMagHeading.Text  := Format('m: %5.1f°', [fMagAccelFusion.fTCMagHeading]); 
        labTrueHeading.Text := Format('t: %5.1f°', [fMagAccelFusion.fTCTrueHeading]);
        labAltitude.Text  := Format('%5.1f°', [fMagAccelFusion.fAltitude] );
        labRoll.Text      := Format('%5.1f°', [fMagAccelFusion.fRoll] );
        ....
        ...
        
* On FormActivate: Start sensors. For Android, you have to ask permission to use the sensors first and start when the permissions are granted. The sample code uses DelphiWorld's permisson requester for that.
For iOS on Delphi 10.4.1 you cannot start the location sensor from FormActivate. I fixed that by activating the sensors from a 2 seconds Timer.  


        procedure TffmMain.timerStartSensorsiOSTimer(Sender: TObject);
        begin
           fMagAccelFusion.StartStopSensors({bStart:} true );  //start ios sensor feed
           timerStartSensorsiOS.Enabled := false;              //once
        end;
        
        procedure TffmMain.FormActivate(Sender: TObject);
        begin
          {$IFDEF Android}  // request permissions to work
          FRequester.RequestPermissions([cPermissionAccessCoarseLocation,cPermissionAccessFineLocation], cPermissionsSensors); 
          {$ENDIF Android}
          
          {$IFDEF IOS}
          // for IOS I found u cannot start LocationSensor from FormActivate or the sensor breaks
          // used a Timer to defer sensor start a couple seconds
          timerStartSensorsiOS.Enabled := true;
          {$ENDIF IOS}
          ...
        end; 
        
        {$IFDEF Android}      // Android requires permissions for things like sensors
        procedure TFormPlanetFun.PermissionsResultHandler(Sender: TObject; const ARequestCode: Integer; const AResults: TPermissionResults);
        var LDeniedResults: TPermissionResults; LDeniedPermissions: string; i:integer;
        begin
          case ARequestCode of     
             cPermissionsSensors: begin
                if AResults.AreAllGranted then  //all granted, start sensors 
                begin
                   fMagAccelFusion.StartStopSensors({bStart:} true );  
                end
                else begin   // denied permissions ? wtf ??
                   LDeniedPermissions := '';
                   LDeniedResults := AResults.DeniedResults;
                   for I := 0 to LDeniedResults.Count - 1 do LDeniedPermissions := LDeniedPermissions + ', ' + LDeniedResults[I].Permission;
                   showToastMessage('You denied permissons ' + LDeniedPermissions + '. We need those!');
                end;   
             end;
          end;  
        end;
       {$ENDIF Android}
          

        
* It is good practice to disable the sensors when the app goes to background (Home btn), and enable when it comes back.       

## Samples
* SensorFusionDemo1 - Simple usage sample in this repository.
* BoatAttitude - A more elaborate sample. A boat 3d model is targeted by a camera controlled by phone attitude. Can be found at https://github.com/omarreis/BoatAttitude . The app illustrates how to use *quaternions* to set 3d object rotations, instead of manipulating RotationAngle. 
* Also in the same repository: sample *AirlinerAttitude* features an 3d airplane model, for the aviation inclined.
.
## SensorFusionDemo screenshot.

![Screenshot](SensorFusionShot.png)

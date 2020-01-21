# Project_on_OCR

TEXT or CHARACTER RECOGNITION SYSTEM. 
This process is also called DOCUMENT IMAGE ANALYSIS (DIA). 
Thus our need is to develop character recognition software system to perform 
Document Image Analysis which transforms documents in paper format to electronic format. 
For this process there are various techniques in the world. 
Among all those techniques we have chosen Optical Character Recognition as main fundamental technique to recognize characters.
The conversion of paper documents into electronic format is an on-going task in many of the organizations particularly in Research and 
Development (R&D) area, in large business enterprises, in government institutions, so on. 
From our problem statement we can introduce the necessity of Optical Character Recognition in mobile electronic 
devices such as cell phones, digital cameras to acquire images and recognize them as a part of face recognition and validation.
Using the Code:-
	The project is developed on a Windows pc 8.1, using android studio IDE.
  1. Preparing Tesseract:-
=> Install the tesseract source code from github.                         (https://github.com/rmtheis/tess-two/)
=> Extract content in a tesseract folder
=> Requires Android 2.2 or higher
=> Download a version 3.02 trained data for a language (English data   for example).
To import tesseract to your android project, we must build it first:-
•	We must have the Android NDK,  which can be download from the given link:- https://dl.google.com/android/ndk/android-ndk-r10c-windows-x86_64.exe
•	After installing the Android NDK,  we must add its install directory to the environment variables under Path
•	To do that we have to go to Control Panel\System and Security\System – advanced system settings – environment variables
•	After adding the android directory to the path. We can use NDK command in the cmd.exe in other directory.
•	Now build the tesseract OCR library using the cmd window.
-	To do these go to the tess-two folder and open cmd window. (Press left shift + Right click).
-	Build the project using:
ndk-build	
android update project –path F:\tess\tess-two
ant release

2. Adding Tess-Two to Android Studio Project:-
	After we have build the tess-two library project, we must import it to the android application project in android studio.
•	In our android studio project tree, add a new directory “libs”, then add a subdirectory name it “tess-two”.
•	In windows explorer, move the content of the tess-two build project to the tess-two directory in libraries android studio.
•	We must add a build.gradle [new file] in the libs\tess-two folder.
-	Make sure all build.gradle files in application project have same target SDK version.
-	Make sure that the tess-two library has build.gradle file.

buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:0.14.0'
    }
}
apply plugin: 'com.android.library'
android {
    compileSdkVersion 22
    buildToolsVersion "22.0.1"
       defaultConfig {
        minSdkVersion 8
        targetSdkVersion 10
    }
        sourceSets.main {
        manifest.srcFile 'AndroidManifest.xml'
        java.srcDirs = ['src']
        resources.srcDirs = ['src']
        res.srcDirs = ['res']
        jniLibs.srcDirs = ['libs']
    }
}
•	Next add to the main settings.gradle include the tess-two library:
include ':app', ':tess-two'
include ':libraries:tess-two'
project(':tess-two').projectDir = new File('libs/tess-two')
•	Next add the tess-two as module dependency to the app module in project structure (crtl+alt+shift+s)

3. Now the tesseract library can be used in our android project:-
	public String detectText(Bitmap bitmap) {
	TessDataManager.initTessTrainedData(context);
  	TessBaseAPI tessBaseAPI = new TessBaseAPI();
	String path = "/mnt/sdcard/packagename/tessdata/eng.traineddata";
	tessBaseAPI.setDebug(true);
	tessBaseAPI.init(path, "eng"); //Init the Tess with the trained data    file, with english language

   //For example if we want to only detect numbers
    tessBaseAPI.setVariable(TessBaseAPI.VAR_CHAR_WHITELIST, "1234567890");
    tessBaseAPI.setVariable(TessBaseAPI.VAR_CHAR_BLACKLIST, "!@#$%^&*()_+=-qwertyuiop[]}{POIU" +
            "YTREWQasdASDfghFGHjklJKLl;L:'\"\\|~`xcvXCVbnmBNM,./<>?");
    	tessBaseAPI.setImage(bitmap);
String text = tessBaseAPI.getUTF8Text();
Log.d(TAG, "Got data: " + result);
tessBaseAPI.end();
return text;
}
4. Android side:-
=> Still we have to take a photo from the camera. We will make a Camera package that loads the camera hardware, and show live streaming on a SurfaceView.
In the CameraManager:  In this class it will invoke the camera hardware by “import android.hardware.Camera;”
We will need a Rect, that will represented to focus box and change it dimension on the event, 
after that when the onDraw is called it will draw the focus box rectangle (design, frame, border and corners) where the cropped photo will take place.

package com.camdiad.characterrecognizer.camera;

import android.content.Context;
import android.content.SharedPreferences;
import android.graphics.Point;
import android.graphics.Rect;
import android.hardware.Camera;
import android.os.Handler;
import android.preference.PreferenceManager;
import android.util.Log;
import android.view.SurfaceHolder;
import com.camdiad.characterrecognizer.PlanarYUVLuminanceSource;
import com.camdiad.characterrecognizer.PreferencesActivity;

import java.io.IOException;


public final class CameraManager {

  private static final String TAG = CameraManager.class.getSimpleName();
  
  private static final int MIN_FRAME_WIDTH = 50; // originally 240
  private static final int MIN_FRAME_HEIGHT = 20; // originally 240
  private static final int MAX_FRAME_WIDTH = 800; // originally 480
  private static final int MAX_FRAME_HEIGHT = 600; // originally 360
  
  private final Context context;
  private final CameraConfigurationManager configManager;
  private Camera camera;
  private AutoFocusManager autoFocusManager;
  private Rect framingRect;
  private Rect framingRectInPreview;
  private boolean initialized;
  private boolean previewing;
  private boolean reverseImage;
  private int requestedFramingRectWidth;
  private int requestedFramingRectHeight;
  /**
   * Preview frames are delivered here, which we pass on to the registered handler. Make sure to
   * clear the handler so it will only receive one message.
   */
  private final PreviewCallback previewCallback;

  public CameraManager(Context context) {
    this.context = context;
    this.configManager = new CameraConfigurationManager(context);
    previewCallback = new PreviewCallback(configManager);
  }

  /**
   * Opens the camera driver and initializes the hardware parameters. 
   */
  public synchronized void openDriver(SurfaceHolder holder) throws IOException {
    Camera theCamera = camera;
    if (theCamera == null) {
      theCamera = Camera.open();
      if (theCamera == null) {
        throw new IOException();
      }
      camera = theCamera;
    }
    camera.setPreviewDisplay(holder);
    if (!initialized) {
      initialized = true;
      configManager.initFromCameraParameters(theCamera);
      if (requestedFramingRectWidth > 0 && requestedFramingRectHeight > 0) {
        adjustFramingRect(requestedFramingRectWidth, requestedFramingRectHeight);
        requestedFramingRectWidth = 0;
        requestedFramingRectHeight = 0;
      }
    }
    configManager.setDesiredCameraParameters(theCamera);
    
    SharedPreferences prefs = PreferenceManager.getDefaultSharedPreferences(context);
    reverseImage = prefs.getBoolean(PreferencesActivity.KEY_REVERSE_IMAGE, false);
  }

  /**
   * Closes the camera driver if still in use.
   */
  public synchronized void closeDriver() {
    if (camera != null) {
      camera.release();
      camera = null;

      // Make sure to clear these each time we close the camera, so that any scanning rect
      // requested by intent is forgotten.
      framingRect = null;
      framingRectInPreview = null;
    }
  }

  /**
   * Asks the camera hardware to begin drawing preview frames to the screen.
   */
  public synchronized void startPreview() {
    Camera theCamera = camera;
    if (theCamera != null && !previewing) {
      theCamera.startPreview();
      previewing = true;
      autoFocusManager = new AutoFocusManager(context, camera);
    }
  }

  /**
   * Tells the camera to stop drawing preview frames.
   */
  public synchronized void stopPreview() {
    if (autoFocusManager != null) {
       autoFocusManager.stop();
       autoFocusManager = null;
    }
   if (camera != null && previewing) {
      camera.stopPreview();
      previewCallback.setHandler(null, 0);
      previewing = false;
    }
  }

  /**
   * A single preview frame will be returned to the handler supplied. The data will arrive as byte[]
   * in the message.obj field, with width and height encoded as message.arg1 and message.arg2,
   * respectively. 
   */
  public synchronized void requestOcrDecode(Handler handler, int message) {
    Camera theCamera = camera;
    if (theCamera != null && previewing) {
      previewCallback.setHandler(handler, message);
      theCamera.setOneShotPreviewCallback(previewCallback);
    }
  }
  
  /**
   * Asks the camera hardware to perform an autofocus.
   */
  public synchronized void requestAutoFocus(long delay) {
   autoFocusManager.start(delay);
  }
  
  /**
   * Calculates the framing rect which the UI should draw to show the user where to place the
   * barcode. This target helps with alignment as well as forces the user to hold the device
   * far enough away to ensure the image will be in focus.
   */
  public synchronized Rect getFramingRect() {
    if (framingRect == null) {
      if (camera == null) {
        return null;
      }
      Point screenResolution = configManager.getScreenResolution();
      if (screenResolution == null) {
        // Called early, before init even finished
        return null;
      }
      int width = screenResolution.x * 3/5;
      if (width < MIN_FRAME_WIDTH) {
        width = MIN_FRAME_WIDTH;
      } else if (width > MAX_FRAME_WIDTH) {
        width = MAX_FRAME_WIDTH;
      }
      int height = screenResolution.y * 1/5;
      if (height < MIN_FRAME_HEIGHT) {
        height = MIN_FRAME_HEIGHT;
      } else if (height > MAX_FRAME_HEIGHT) {
        height = MAX_FRAME_HEIGHT;
      }
      int leftOffset = (screenResolution.x - width) / 2;
      int topOffset = (screenResolution.y - height) / 2;
      framingRect = new Rect(leftOffset, topOffset, leftOffset + width, topOffset + height);
    }
    return framingRect;
  }

  /**
   * Like {@link #getFramingRect} but coordinates are in terms of the preview frame,
   * not UI / screen.
   */
  public synchronized Rect getFramingRectInPreview() {
    if (framingRectInPreview == null) {
      Rect rect = new Rect(getFramingRect());
      Point cameraResolution = configManager.getCameraResolution();
      Point screenResolution = configManager.getScreenResolution();
      if (cameraResolution == null || screenResolution == null) {
        // Called early, before init even finished
        return null;
      }
      rect.left = rect.left * cameraResolution.x / screenResolution.x;
      rect.right = rect.right * cameraResolution.x / screenResolution.x;
      rect.top = rect.top * cameraResolution.y / screenResolution.y;
      rect.bottom = rect.bottom * cameraResolution.y / screenResolution.y;
      framingRectInPreview = rect;
    }
    return framingRectInPreview;
  }

  /**
   * Changes the size of the framing rect. 
   */
  public synchronized void adjustFramingRect(int deltaWidth, int deltaHeight) {
    if (initialized) {
      Point screenResolution = configManager.getScreenResolution();

      // Set maximum and minimum sizes
      if ((framingRect.width() + deltaWidth > screenResolution.x - 4) || (framingRect.width() + deltaWidth < 50)) {
        deltaWidth = 0;
      }
      if ((framingRect.height() + deltaHeight > screenResolution.y - 4) || (framingRect.height() + deltaHeight < 50)) {
        deltaHeight = 0;
      }

      int newWidth = framingRect.width() + deltaWidth;
      int newHeight = framingRect.height() + deltaHeight;
      int leftOffset = (screenResolution.x - newWidth) / 2;
      int topOffset = (screenResolution.y - newHeight) / 2;
      framingRect = new Rect(leftOffset, topOffset, leftOffset + newWidth, topOffset + newHeight);
      framingRectInPreview = null;
    } else {
      requestedFramingRectWidth = deltaWidth;
      requestedFramingRectHeight = deltaHeight;
    }
  }

  /**
   * A factory method to build the appropriate LuminanceSource object based on the format
   * of the preview buffers, as described by Camera.Parameters. 
   */
  public PlanarYUVLuminanceSource buildLuminanceSource(byte[] data, int width, int height) {
    Rect rect = getFramingRectInPreview();
    if (rect == null) {
      return null;
    }
    // Go ahead and assume it's YUV rather than die.
    return new PlanarYUVLuminanceSource(data, width, height, rect.left, rect.top,
                                        rect.width(), rect.height(), reverseImage);
  }

}

=> For the focus box view, we have to create an AutoFocusManager class that implements Camera.AutoFocusCallback. 

package com.camdiad.characterrecognizer.camera;

import android.content.Context;
import android.content.SharedPreferences;
import android.hardware.Camera;
import android.preference.PreferenceManager;
import android.util.Log;
import com.camdiad.characterrecognizer.PreferencesActivity;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Timer;
import java.util.TimerTask;

public final class AutoFocusManager implements Camera.AutoFocusCallback {

  private static final String TAG = AutoFocusManager.class.getSimpleName();

  private static final long AUTO_FOCUS_INTERVAL_MS = 3500L;
  private static final Collection<String> FOCUS_MODES_CALLING_AF;
  static {
    FOCUS_MODES_CALLING_AF = new ArrayList<String>(2);
    FOCUS_MODES_CALLING_AF.add(Camera.Parameters.FOCUS_MODE_AUTO);
    FOCUS_MODES_CALLING_AF.add(Camera.Parameters.FOCUS_MODE_MACRO);
  }

  private boolean active;
  private boolean manual;
  private final boolean useAutoFocus;
  private final Camera camera;
  private final Timer timer;
  private TimerTask outstandingTask;

  AutoFocusManager(Context context, Camera camera) {
    this.camera = camera;
    timer = new Timer(true);
    SharedPreferences sharedPrefs = PreferenceManager.getDefaultSharedPreferences(context);
    String currentFocusMode = camera.getParameters().getFocusMode();
    useAutoFocus =
        sharedPrefs.getBoolean(PreferencesActivity.KEY_AUTO_FOCUS, true) &&
        FOCUS_MODES_CALLING_AF.contains(currentFocusMode);
    Log.i(TAG, "Current focus mode '" + currentFocusMode + "'; use auto focus? " + useAutoFocus);
    manual = false;
    checkAndStart();
  }

  @Override
  public synchronized void onAutoFocus(boolean success, Camera theCamera) {
    if (active && !manual) {
      outstandingTask = new TimerTask() {
        @Override
        public void run() {
          checkAndStart();
        }
      };
      timer.schedule(outstandingTask, AUTO_FOCUS_INTERVAL_MS);
    }
    manual = false;
  }

  void checkAndStart() {
   if (useAutoFocus) {
     active = true;
      start();
    }
  }

  synchronized void start() {
     try {
        camera.autoFocus(this);
     } catch (RuntimeException re) {
        // Have heard RuntimeException reported in Android 4.0.x+; continue?
        Log.w(TAG, "Unexpected exception while focusing", re);
     }
  }

  /**
   * Performs a manual auto-focus after the given delay.
   */
  synchronized void start(long delay) {
   outstandingTask = new TimerTask() {
      @Override
      public void run() {
         manual = true;
         start();
      }
   };
   timer.schedule(outstandingTask, delay);
  }

  synchronized void stop() {
    if (useAutoFocus) {
      camera.cancelAutoFocus();
    }
    if (outstandingTask != null) {
      outstandingTask.cancel();
      outstandingTask = null;
    }
    active = false;
    manual = false;
  }

}

=> Now we need a button designed to be used for the on-screen shutter button. It's currently an {@code ImageView} that can call a delegate when the pressed state changes.
package com.camdiad.characterrecognizer.camera;

import android.content.Context;
import android.util.AttributeSet;
import android.view.SoundEffectConstants;
import android.widget.ImageView;

public class ShutterButton extends ImageView {
   /**
    * A callback to be invoked when a ShutterButton's pressed state changes.
    */
   public interface OnShutterButtonListener {
      /**
       * Called when a ShutterButton has been pressed. 
       */
      void onShutterButtonFocus(ShutterButton b, boolean pressed);

      void onShutterButtonClick(ShutterButton b);
   }

   private OnShutterButtonListener mListener;
   private boolean mOldPressed;

   public ShutterButton(Context context) {
      super (context);
   }

   public ShutterButton(Context context, AttributeSet attrs) {
      super (context, attrs);
   }

   public ShutterButton(Context context, AttributeSet attrs,
         int defStyle) {
      super (context, attrs, defStyle);
   }

   public void setOnShutterButtonListener(OnShutterButtonListener listener) {
      mListener = listener;
   }

   /**
    * Hook into the drawable state changing to get changes to isPressed -- the
    * onPressed listener doesn't always get called when the pressed state
    * changes.
    */
    @Override
    protected void drawableStateChanged() {
       super .drawableStateChanged();
       final boolean pressed = isPressed();
       if (pressed != mOldPressed) {
          if (!pressed) {
             
             post(new Runnable() {
                public void run() {
                   callShutterButtonFocus(pressed);
                }
             });
          } else {
             callShutterButtonFocus(pressed);
          }
          mOldPressed = pressed;
       }
    }

    private void callShutterButtonFocus(boolean pressed) {
       if (mListener != null) {
          mListener.onShutterButtonFocus(this , pressed);
       }
    }

    @Override
    public boolean performClick() {
       boolean result = super.performClick();
       playSoundEffect(SoundEffectConstants.CLICK);
       if (mListener != null) {
          mListener.onShutterButtonClick(this);
       }
       return result;
    }
}

=> Now our next task is to create a package name called “language”. In this package it contains the classes that translate the recognize text into chosen language (For example Hindi).
- Translator.java:- This class contains the delegate’s translation requests to the appropriate translation service (For example Google translator and Bing translator).

package com.camdiad.characterrecognizer.language;

import android.app.Activity;
import android.content.SharedPreferences;
import android.preference.PreferenceManager;

import com.camdiad.characterrecognizer.CaptureActivity;
import com.camdiad.characterrecognizer.PreferencesActivity;

/**
 * Delegates translation requests to the appropriate translation service.
 */
public class Translator {

  public static final String BAD_TRANSLATION_MSG = "[Translation unavailable]";
  
  private Translator(Activity activity) {  
    // Private constructor to enforce noninstantiability
  }
  
  static String translate(Activity activity, String sourceLanguageCode, String targetLanguageCode, String sourceText) {   
    
    // Check preferences to determine which translation API to use--Google, or Bing.
    SharedPreferences prefs = PreferenceManager.getDefaultSharedPreferences(activity);
    String api = prefs.getString(PreferencesActivity.KEY_TRANSLATOR, CaptureActivity.DEFAULT_TRANSLATOR);
    
    // Delegate the translation based on the user's preference.
    if (api.equals(PreferencesActivity.TRANSLATOR_BING)) {
      
      // Get the correct code for the source language for this translation service.
      sourceLanguageCode = TranslatorBing.toLanguage(
          LanguageCodeHelper.getTranslationLanguageName(activity.getBaseContext(), sourceLanguageCode));
      
      return TranslatorBing.translate(sourceLanguageCode, targetLanguageCode, sourceText);
    } else if (api.equals(PreferencesActivity.TRANSLATOR_GOOGLE)) {
      
      // Get the correct code for the source language for this translation service.
      sourceLanguageCode = TranslatorGoogle.toLanguage(
          LanguageCodeHelper.getTranslationLanguageName(activity.getBaseContext(), sourceLanguageCode));      
      
      return TranslatorGoogle.translate(sourceLanguageCode, targetLanguageCode, sourceText);
    }
    return BAD_TRANSLATION_MSG;
  }
}

- TranslatorBing.java:- This class will invoke the Microsoft Bing Translator API. Which helps to translate the recognize characters.
package com.camdiad.characterrecognizer.language;

import android.util.Log;
import com.memetix.mst.language.Language;
import com.memetix.mst.translate.Translate;
import com.camdiad.characterrecognizer.CaptureActivity;

public class TranslatorBing {
  private static final String TAG = TranslatorBing.class.getSimpleName();
  private static final String CLIENT_ID = " Character_Recognizer ";
  private static final String CLIENT_SECRET = " /5Za8pysr3Oosl1x96Ry7pL9KVRNZZGyRwnlwNJafQQ= ";
  
  /**
   *  Translate using Microsoft Translate API 
   */
  static String translate(String sourceLanguageCode, String targetLanguageCode, String sourceText) {
    Translate.setClientId(CLIENT_ID);
    Translate.setClientSecret(CLIENT_SECRET);
    try {
      Log.d(TAG, sourceLanguageCode + " -> " + targetLanguageCode);
      return Translate.execute(sourceText, Language.fromString(sourceLanguageCode), 
          Language.fromString(targetLanguageCode));
    } catch (Exception e) {
      Log.e(TAG, "Caught exeption in translation request.");
      e.printStackTrace();
      return Translator.BAD_TRANSLATION_MSG;
    }
  }
  
  /**
   * Convert the given name of a natural language into a Language from the enum of Languages 
   * supported by this translation service. 
   */
  public static String toLanguage(String languageName) throws IllegalArgumentException {    
    // Convert string to all caps
    String standardizedName = languageName.toUpperCase();
    
    // Replace spaces with underscores
    standardizedName = standardizedName.replace(' ', '_');
    
    // Remove parentheses
    standardizedName = standardizedName.replace("(", "");   
    standardizedName = standardizedName.replace(")", "");
    
    // Map Norwegian-Bokmal to Norwegian
    if (standardizedName.equals("NORWEGIAN_BOKMAL")) {
      standardizedName = "NORWEGIAN";
    }
    
    try {
      return Language.valueOf(standardizedName).toString();
    } catch (IllegalArgumentException e) {
      Log.e(TAG, "Not found--returning default language code");
      return CaptureActivity.DEFAULT_TARGET_LANGUAGE_CODE;
    }
  }
}

- TranslatorGoogle.java:- This class will invoke the Google Translator setvice. By using the “API_KEY”.
package com.camdiad.characterrecognizer.language;

import android.util.Log;
import com.google.api.GoogleAPI;
import com.google.api.translate.Language;
import com.google.api.translate.Translate;

import com.camdiad.characterrecognizer.CaptureActivity;

public class TranslatorGoogle {
  private static final String TAG = TranslatorGoogle.class.getSimpleName();
  private static final String API_KEY = " AIzaSyC22o1wkCLA8Kk8WnGQ9a_AhbU_1R_jcqg ";

  private TranslatorGoogle() {  
    // Private constructor to enforce noninstantiability
  }

  // Translate using Google Translate API
  static String translate(String sourceLanguageCode, String targetLanguageCode, String sourceText) {   
    Log.d(TAG, sourceLanguageCode + " -> " + targetLanguageCode);
    
    // Truncate excessively long strings. Limit for Google Translate is 5000 characters
    if (sourceText.length() > 4500) {
      sourceText = sourceText.substring(0, 4500);
    }
    
    GoogleAPI.setKey(API_KEY);
    GoogleAPI.setHttpReferrer("https://github.com/rmtheis/android-ocr");
    try {
      return Translate.DEFAULT.execute(sourceText, Language.fromString(sourceLanguageCode), 
          Language.fromString(targetLanguageCode));
    } catch (Exception e) {
      Log.e(TAG, "Caught exeption in translation request.");
      return Translator.BAD_TRANSLATION_MSG;
    }
  }

  /**
   * Convert the given name of a natural language into a language code from the enum of Languages 
   * supported by this translation service. 
   */
  public static String toLanguage(String languageName) throws IllegalArgumentException {   
    // Convert string to all caps
    String standardizedName = languageName.toUpperCase();
    
    // Replace spaces with underscores
    standardizedName = standardizedName.replace(' ', '_');
    
    // Remove parentheses
    standardizedName = standardizedName.replace("(", "");   
    standardizedName = standardizedName.replace(")", "");
    
    // Hack to fix misspelling in google-api-translate-java
    if (standardizedName.equals("UKRAINIAN")) {
      standardizedName = "UKRANIAN";
    }
    
    // Map Norwegian-Bokmal to Norwegian
    if (standardizedName.equals("NORWEGIAN_BOKMAL")) {
      standardizedName = "NORWEGIAN";
    }
    
    try {
      return Language.valueOf(standardizedName).toString();
    } catch (IllegalArgumentException e) {
      Log.e(TAG, "Not found--returning default language code");
      return CaptureActivity.DEFAULT_TARGET_LANGUAGE_CODE;
    }
  }
}

=> In this Android project I have use three activities along with their Layout xml files they are:-
-	CaptureActivity.java
     - capture.xml 
-	HelpActivity.java
     - help.xml
-	PreferencesActivity.java	
CaptureActivity.java:- This activity opens the camera and does the actual scanning on a background thread. It draws a viewfinder to help the user place the text correctly, shows feedback as the image processing is happing, and then overlays the results when a scan is successful.
This code for this class was adapted from the ZXing project: http://code.google.com/p/zxing

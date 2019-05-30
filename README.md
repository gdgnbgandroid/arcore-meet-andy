# Meet Andy: ARCore Workshop
You are going to create a small app that uses ARCore to place Andy the green robot into the real world. You can then take pictures of those scenes and safe them to your storage.

## Step 0
* Install Android Studio 3.1 or greater
* [Enable developer options](https://developer.android.com/studio/debug/dev-options#enable) on your device
* Install the SceneForm plugin: File -> Settings -> Plugins -> Search for "SceneForm"

## Step 1.a: Create the project
* File -> New -> New Project
* Add No Activity
* Name "Meet Andy"
* Language "Kotlin"
* Minimum API Level for ARCore: 24
* Use androidx.* artifacts for nicer dependencies

## Step 1.b: Setup SDK and emulator
* Tools -> SDK Manager
* Install at least Android 7.0 (API level 24), better choose Pie or Q
* If your device is compatible, you can skip emulator setup, compatible devices: bit.ly/arcore-devices
* If not compatible: Tools -> AVD Manager
* Click Create Virtual Device
* Choose e.g. Pixel 3 device definition
* Download a system image with at least API level 24
* We need OpenGL 3.0 or higher and Android Emulator 27 or higher for SceneForm
* Tools -> SDK Manager -> SDK Tools -> Check Android Emulator Version
* Start your emulator, then check through adb which Open GL version is used: adb logcat | grep eglMakeCurrent
* If it's below 3.0, enforce through Extended Controls -> Settings -> Advanced
* If using emulator: Configure your cameras, back camera Virtual Scene and / or front camera webcam0 (if available)
* If using emulator: update ARCore
* Go to bit.ly/arcore-apk
* Download the latest ARCore for emulator APK
* Sideload it to your emulator with
```
adb install -r ARCore_*_x86_for_emulator.apk
```

## Step 2: Configure the project
* `AndroidManifest.xml`: Tell Android that your app needs ARCore compatibility to run. Add the following `meta-data` tag inside `application`. If users device does not already have ARCore on device, PlayStore will download it.
``` lang=xml
<application
    ... >
    <meta-data android:name="com.google.ar.core" android:value="required" />
</application>
```
* `AndroidManifest.xml`: We also have to request access to some needed features and permissions. Add the following inside the manifest tag.
``` lang=xml
<manifest
    ... >
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-feature android:name="android.hardware.camera.ar" />
</manifest>
```
* Module level `build.gradle`: SceneForm needs Java 8 language features. As we're targeting SDK lvls below 27, so we need to ensure source compatibility. Add this to the `android` section.
``` lang=groovy
android {
    ...
    compileOptions {
            sourceCompatibility 1.8
            targetCompatibility 1.8
    }
}
```
* Module level `build.gradle`: Now add the dependencies for SceneForm and ARCore to the `dependencies` section
``` lang=groovy
dependencies {
    …
    implementation 'com.google.ar:core:1.9.0'
    implementation 'com.google.ar.sceneform.ux:sceneform-ux:1.9.0'
}
```

## Step 3.a: Create an ArFragment
Ideally you should check if the device you are running on is compatible with the features you need and request necessary permissions during runtime. `ArFragment` does all this for you.
1. Checks whether a compatible version of ARCore is installed, prompting the user to install or update as necessary
2. Checks whether the app has access to the camera, and asks the user for permission if it has not yet been granted

* Choose File -> New -> Android Activity -> Empty Activity
* Call it MainActivity, check "Launcher Activity"
* Open res/layout/activity_main.xml and replace its content with
``` lang=xml
<?xml version="1.0" encoding="utf-8"?>
<fragment
        xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        tools:context=".MainActivity"
        android:name="com.google.ar.sceneform.ux.ArFragment"
        android:id="@+id/ux_fragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
```

## Step 3.b: Run it!
You should now be able to start the app and walk around until planes are detected.

## Step 4: Invite Andy to the scene
* Andy comes in the form of a `Renderable`, an abstraction above Meshes, Materials and Textures.
* In Android Studio, click on the `app` folder and choose New -> Sample Data Directory
* A good source for renderables is poly.google.com, search for "Andy"
* Download the OBJ variant of Andy
* Extract everything to the models folder you just created (right click on it, then choose "Show in Files" if you can't find it)
* Back in Android Studio, right click on the OBJ file -> Click "Import SceneForm Asset", accept the defaults
* Then change your MainActivity to this:
``` lang=kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var andyRenderable: Renderable

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        ModelRenderable.builder()
            .setSource(this, Uri.parse("Andy.sfb"))
            .build()
            .thenAccept { renderable: Renderable -> andyRenderable = renderable }
            .exceptionally {
                val toast = Toast.makeText(this, "Unable to load andy renderable", Toast.LENGTH_LONG)
                toast.setGravity(Gravity.CENTER, 0, 0)
                toast.show()
                null
            }

    }
}
```
* To tell your compiler that the ux_fragment is of type ArFragment add this to the top of `MainActivity`
``` lang=kotlin
private lateinit var arFragment: ArFragment
```
* Then in `MainActivity.onCreate()` add the initialization:
``` lang=kotlin
arFragment = ux_fragment as ArFragment
```
* There's more to add: a `TapArPlaneListener` right after the model builder to detect if the user tapped on a plane. If the listener is fired, we add Andy to the scene!
``` lang=kotlin
arFragment.setOnTapArPlaneListener { hitResult, plane, motionEvent ->
            // Create the Anchor.
            val anchor = hitResult.createAnchor()
            val anchorNode = AnchorNode(anchor)
            anchorNode.setParent(arFragment.arSceneView.scene)

            // Create the transformable andy and add it to the anchor.
            val andy = TransformableNode(arFragment.transformationSystem)
            andy.setParent(anchorNode)
            andy.renderable = andyRenderable
            andy.select()
        }
```
* Run the app and play around

## Step 5: Save your pictures with Andy
* Open `res/activity_main.xml`, we'll add two FloatingActionButtons to change between front and back camera and to take a picture. Replace the whole content of the file with the following:
``` lang=xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <fragment
        android:id="@+id/ux_fragment"
        android:name="com.google.ar.sceneform.ux.ArFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context=".MainActivity" />

    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/fabTakePicture"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginEnd="16dp"
        android:layout_marginBottom="16dp"
        android:clickable="true"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:srcCompat="@android:drawable/ic_menu_camera"
        android:focusable="true" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

* Add a dependency to Android Material to your Module level `build.gradle`:
``` lang=json
dependencies {
    implementation 'com.google.android.material:material:1.0.0'
}
```

* We now need to request permission to write files to external storage
* Right-Click on your `res` folder, New -> Android Ressource directory -> xml 
* Inside this folder, create the file `paths.xml` (New -> XML Ressource Fiel) and put this inside:
``` lang=xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
   <external-path name="meet-andy" path="Pictures" />
</paths>
```
* Our manifest `AndroidManifest.xml` needs to know about the path, needs to inform about the needed permission to write to storage and we'll also add an Url provider to safely share URIs with other apps
``` lang=xml
<manifest>
...
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
<application>
    ...
    <provider
        android:name="androidx.core.content.FileProvider"
        android:authorities="${applicationId}.meetandy.name.provider"
        android:exported="false"
        android:grantUriPermissions="true">
        <meta-data
            android:name="android.support.FILE_PROVIDER_PATHS"
            android:resource="@xml/paths"/>
    </provider>
</application>
</manifest>
```

* Permissions are actually requested at runtime and until know our ArFragment took care about this. We need to extend it to request more permissions.
* Create a new Kotlin class file, name it `PhotoArFragment.kt` and put this inside:
``` lang=kotlin
class PhotoArFragment : ArFragment() {
    override fun getAdditionalPermissions(): Array<String?> {
        val additionalPermissions = super.getAdditionalPermissions()
        val permissionLength = additionalPermissions?.size ?: 0
        val permissions = arrayOfNulls<String>(permissionLength + 1)
        permissions[0] = Manifest.permission.WRITE_EXTERNAL_STORAGE
        if (permissionLength > 0) {
            System.arraycopy(additionalPermissions!!, 0, permissions, 1, additionalPermissions.size)
        }
        return permissions
    }
}
```

* Right-Click on the class name -> Copy reference
* Open `res/layout/activity_main.xml` and replace the attribute android:name of the fragment with the reference to `PhotoArFragment`
* To take photos and open it with the photos app, we need to add a bit of boilerplate code to our `PhotoArFragment`, just replace the class with the following:
``` lang=kotlin
class PhotoArFragment : ArFragment() {
    override fun getAdditionalPermissions(): Array<String?> {
        val additionalPermissions = super.getAdditionalPermissions()
        val permissionLength = additionalPermissions?.size ?: 0
        val permissions = arrayOfNulls<String>(permissionLength + 1)
        permissions[0] = Manifest.permission.WRITE_EXTERNAL_STORAGE
        if (permissionLength > 0) {
            System.arraycopy(additionalPermissions!!, 0, permissions, 1, additionalPermissions.size)
        }
        return permissions
    }

    private fun generateFilename(): String {
        val date = SimpleDateFormat("yyyyMMdd-HHmmss", Locale.getDefault()).format(Date())
        return Environment.getExternalStoragePublicDirectory(
            Environment.DIRECTORY_PICTURES
        ).toString() + File.separator + "MeetAndy/" + date + ".jpg"
    }

    @Throws(IOException::class)
    private fun saveBitmapToDisk(bitmap: Bitmap, filename: String) {
        val out = File(filename)
        if (!out.parentFile.exists()) {
            out.parentFile.mkdirs()
        }
        try {
            FileOutputStream(filename).use { outputStream ->
                ByteArrayOutputStream().use { outputData ->
                    bitmap.compress(Bitmap.CompressFormat.PNG, 100, outputData)
                    outputData.writeTo(outputStream)
                    outputStream.flush()
                    outputStream.close()
                }
            }
        } catch (ex: IOException) {
            throw IOException("Failed to save bitmap to disk", ex)
        }
    }

    fun takePhoto() {
        val filename = generateFilename()

        // Create a bitmap the size of the scene view.
        val bitmap = Bitmap.createBitmap(
            arSceneView.width, arSceneView.height,
            Bitmap.Config.ARGB_8888
        )

        // Create a handler thread to offload the processing of the image.
        val handlerThread = HandlerThread("PixelCopier")
        handlerThread.start()
        // Make the request to copy.
        PixelCopy.request(arSceneView, bitmap, { copyResult: Int ->
            if (copyResult == PixelCopy.SUCCESS) {
                try {
                    saveBitmapToDisk(bitmap, filename)
                } catch (e: IOException) {
                    val toast = Toast.makeText(
                        activity, e.toString(),
                        Toast.LENGTH_LONG
                    )
                    toast.show()
                    return@request
                }

                val snackbar = Snackbar.make(
                    view!!,
                    "Photo saved", Snackbar.LENGTH_LONG
                )
                snackbar.setAction("Open in Photos") { v ->
                    val photoFile = File(filename)

                    val photoURI = FileProvider.getUriForFile(
                        activity!!.applicationContext,
                        activity!!.packageName + ".meetandy.name.provider",
                        photoFile
                    )
                    val intent = Intent(Intent.ACTION_VIEW, photoURI)
                    intent.setDataAndType(photoURI, "image/*")
                    intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
                    startActivity(intent)

                }
                snackbar.show()
            } else {
                val toast = Toast.makeText(
                    activity,
                    "Failed to copyPixels: $copyResult", Toast.LENGTH_LONG
                )
                toast.show()
            }
            handlerThread.quitSafely()
        }, Handler(handlerThread.looper))
    }
}
```
* In our `MainActivity` we change the type of `arFragment` to `PhotoArFragment`
* Now we can easily add a click listener to the photo action button in our `MainActivity.kt`. Just add this to the bottom of the `onCreate()` method:
`fabTakePicture.setOnClickListener { arFragment.takePhoto() }`
* That's it. Run the app and take photos!

## Step 6: Get a picture of you and Andy
* Front camera does not work for now due to too low resolution. Face detection and AugmentedFaces does work with front camera.
* No problem: Add a countdown, then position yourself in the picture
* Add a TextView to our `activity_main.xml`, inside the `ConstraintLayout` tag
``` lang=xml
<ConstraintLayout
...>
    ...
    <TextView
            android:id="@+id/tvCountdown"
            android:layout_width="0dp"
            android:layout_height="48dp"
            android:layout_marginStart="32dp"
            android:layout_marginEnd="32dp"
            android:layout_marginBottom="32dp"
            android:visibility="invisible"
            android:text="5"
            android:textColor="@android:color/white"
            android:textSize="36sp"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toStartOf="@+id/fabTakePicture"
            app:layout_constraintStart_toStartOf="parent" />
</ConstraintLayout>
```
* Go to `MainActivity` and add the countdown logic. We also want to clean up the scene a bit and remove the plane indicator and selection indicator. Replace `MainActivity` with this:
``` lang=kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var andyRenderable: Renderable
    private lateinit var arFragment: PhotoArFragment
    private var selectedAndy: TransformableNode? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        arFragment = ux_fragment as PhotoArFragment

        ModelRenderable.builder()
            .setSource(this, Uri.parse("Andy.sfb"))
            .build()
            .thenAccept { renderable: Renderable -> andyRenderable = renderable }
            .exceptionally {
                val toast = Toast.makeText(this, "Unable to load andy renderable", Toast.LENGTH_LONG)
                toast.setGravity(Gravity.CENTER, 0, 0)
                toast.show()
                null
            }

        arFragment.setOnTapArPlaneListener { hitResult, plane, motionEvent ->
            // Create the Anchor.
            val anchor = hitResult.createAnchor()
            val anchorNode = AnchorNode(anchor)
            anchorNode.setParent(arFragment.arSceneView.scene)

            // Create the transformable andy and add it to the anchor.
            val andy = TransformableNode(arFragment.transformationSystem)
            andy.setParent(anchorNode)
            andy.renderable = andyRenderable
            selectedAndy = andy
            andy.select()
        }

        fabTakePicture.setOnClickListener { takePhotoWithCountdown() }
    }

    private fun takePhotoWithCountdown() {
        object : CountDownTimer(4000, 1000) {

            override fun onTick(millisUntilFinished: Long) {
                tvCountdown.text = ((millisUntilFinished / 1000) + 1).toString()
            }

            override fun onFinish() {
                tvCountdown.text = "Cheese!"
                tvCountdown.postDelayed({
                    tvCountdown.visibility = View.INVISIBLE
                    fabTakePicture.show()
                    arFragment.arSceneView.planeRenderer.isEnabled = true
                }, 1000)
                arFragment.takePhoto()
            }
        }.start()
        // Hide all the chrome for a nice photo
        arFragment.arSceneView.planeRenderer.isEnabled = false
        tvCountdown.visibility = View.VISIBLE
        fabTakePicture.hide()
        selectedAndy?.transformationSystem?.selectNode(null)
    }
}
```

## FINISH
Congratulations,you just created an app that uses Augmented Reality to create a whole new set of experiences for your users.


     import android.Manifest
     import android.content.Intent
     import android.content.pm.PackageManager
     import android.graphics.Bitmap
     import android.os.Bundle
     import android.provider.MediaStore
     import android.widget.Button
     import android.widget.ImageView
     import android.widget.TextView
     import android.widget.Toast
     import androidx.activity.result.contract.ActivityResultContracts
     import androidx.appcompat.app.AppCompatActivity
     import androidx.core.app.ActivityCompat
     import androidx.core.content.ContextCompat
     import com.google.android.gms.vision.Frame
     import com.google.android.gms.vision.text.TextRecognizer
     import retrofit2.Call
     import retrofit2.Callback
     import retrofit2.Response
     import retrofit2.Retrofit
     import retrofit2.converter.gson.GsonConverterFactory
     import retrofit2.http.GET
     import retrofit2.http.Query

     class MainActivity : AppCompatActivity() {

         private lateinit var imageView: ImageView
         private lateinit var resultText: TextView
         private val CAMERA_REQUEST_CODE = 100

         // Retrofit interface untuk Google Custom Search API
         interface GoogleSearchApi {
             @GET("customsearch/v1")
             fun search(
                 @Query("key") apiKey: String,
                 @Query("cx") searchEngineId: String,
                 @Query("q") query: String
             ): Call<SearchResponse>
         }

         data class SearchResponse(val items: List<SearchItem>?)
         data class SearchItem(val title: String, val link: String)

         private val retrofit = Retrofit.Builder()
             .baseUrl("https://www.googleapis.com/")
             .addConverterFactory(GsonConverterFactory.create())
             .build()

         private val api = retrofit.create(GoogleSearchApi::class.java)

         override fun onCreate(savedInstanceState: Bundle?) {
             super.onCreate(savedInstanceState)
             setContentView(R.layout.activity_main)

             imageView = findViewById(R.id.image_view)
             resultText = findViewById(R.id.result_text)
             val btnCapture = findViewById<Button>(R.id.btn_capture)

             btnCapture.setOnClickListener {
                 if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA) != PackageManager.PERMISSION_GRANTED) {
                     ActivityCompat.requestPermissions(this, arrayOf(Manifest.permission.CAMERA), CAMERA_REQUEST_CODE)
                 } else {
                     openCamera()
                 }
             }
         }

         private fun openCamera() {
             val intent = Intent(MediaStore.ACTION_IMAGE_CAPTURE)
             cameraLauncher.launch(intent)
         }

         private val cameraLauncher = registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
             if (result.resultCode == RESULT_OK) {
                 val bitmap = result.data?.extras?.get("data") as Bitmap
                 imageView.setImageBitmap(bitmap)
                 processImage(bitmap)
             }
         }

         private fun processImage(bitmap: Bitmap) {
             // Gunakan TextRecognizer untuk mendeteksi teks (sebagai proxy untuk objek)
             val textRecognizer = TextRecognizer.Builder(applicationContext).build()
             if (!textRecognizer.isOperational) {
                 Toast.makeText(this, "Text recognizer tidak tersedia", Toast.LENGTH_SHORT).show()
                 return
             }

             val frame = Frame.Builder().setBitmap(bitmap).build()
             val textBlocks = textRecognizer.detect(frame)
             val detectedText = StringBuilder()

             for (i in 0 until textBlocks.size()) {
                 detectedText.append(textBlocks[i].value).append(" ")
             }

             if (detectedText.isNotEmpty()) {
                 searchGoogle(detectedText.toString().trim())
             } else {
                 resultText.text = "Tidak ada teks terdeteksi. Coba lagi."
             }
         }

         private fun searchGoogle(query: String) {
             val apiKey = "YOUR_API_KEY_HERE"  // Ganti dengan API key Anda
             val searchEngineId = "YOUR_SEARCH_ENGINE_ID_HERE"  // Ganti dengan Custom Search Engine ID

             api.search(apiKey, searchEngineId, query).enqueue(object : Callback<SearchResponse> {
                 override fun onResponse(call: Call<SearchResponse>, response: Response<SearchResponse>) {
                     if (response.isSuccessful) {
                         val items = response.body()?.items
                         if (!items.isNullOrEmpty()) {
                             val result = items[0].title + "\n" + items[0].link
                             resultText.text = result
                         } else {
                             resultText.text = "Tidak ada hasil pencarian."
                         }
                     } else {
                         resultText.text = "Error: ${response.message()}"
                     }
                 }

                 override fun onFailure(call: Call<SearchResponse>, t: Throwable) {
                     resultText.text = "Gagal menghubungi API: ${t.message}"
                 }
             })
         }

         override fun onRequestPermissionsResult(requestCode: Int, permissions: Array<out String>, grantResults: IntArray) {
             super.onRequestPermissionsResult(requestCode, permissions, grantResults)
             if (requestCode == CAMERA_REQUEST_CODE && grantResults.isNotEmpty() && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                 openCamera()
             } else {
                 Toast.makeText(this, "Izin kamera diperlukan", Toast.LENGTH_SHORT).show()
             }
         }
     }
     

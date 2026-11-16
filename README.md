pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories { google(); mavenCentral() }
}
rootProject.name = "EzanVakitleri"
include(":app")
plugins {
    id("com.android.application") version "8.6.1" apply false
    id("org.jetbrains.kotlin.android") version "2.0.20" apply false
    id("org.jetbrains.kotlin.plugin.compose") version "2.0.20" apply false
}
org.gradle.jvmargs=-Xmx2g -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
}
android {
    namespace = "tr.app.ezanvakitleri"
    compileSdk = 35
    defaultConfig {
        applicationId = "tr.app.ezanvakitleri"
        minSdk = 24
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }
    buildFeatures { compose = true }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions { jvmTarget = "17" }
}
dependencies {
    implementation(platform("androidx.compose:compose-bom:2024.09.03"))
    implementation("androidx.activity:activity-compose:1.9.2")
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.material:material-icons-extended")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.6")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.9.0")
    debugImplementation("androidx.compose.ui:ui-tooling")
}
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET" />
    <application
        android:allowBackup="true"
        android:label="Ezan Vakitleri"
        android:supportsRtl="true"
        android:theme="@android:style/Theme.Material.Light.NoActionBar"
        android:usesCleartextTraffic="false">
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
package tr.app.ezanvakitleri

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.LocationOn
import androidx.compose.material.icons.filled.Refresh
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import org.json.JSONArray
import org.json.JSONObject
import java.net.HttpURLConnection
import java.net.URL
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale
import java.util.TimeZone

private const val API = "https://namazvakti.mtopal.dev"

data class Place(val id: String, val name: String)
data class PrayerDay(
    val date: String, val hijri: String, val imsak: String,
    val gunes: String, val ogle: String, val ikindi: String,
    val aksam: String, val yatsi: String
)

private suspend fun getJson(path: String): String = withContext(Dispatchers.IO) {
    val conn = (URL(API + path).openConnection() as HttpURLConnection)
    conn.connectTimeout = 15000
    conn.readTimeout = 15000
    conn.requestMethod = "GET"
    try {
        val code = conn.responseCode
        val stream = if (code in 200..299) conn.inputStream else conn.errorStream
        val body = stream.bufferedReader(Charsets.UTF_8).use { it.readText() }
        if (code !in 200..299) error("Sunucu hatası ($code)")
        body
    } finally { conn.disconnect() }
}

private fun parsePlaces(json: String, idKey: String, nameKey: String): List<Place> {
    val arr = JSONArray(json)
    return (0 until arr.length()).mapNotNull { i ->
        val o = arr.optJSONObject(i) ?: return@mapNotNull null
        val id = o.optString(idKey).takeIf { it.isNotBlank() && it != "null" } ?: return@mapNotNull null
        val name = o.optString(nameKey).takeIf { it.isNotBlank() && it != "null" } ?: return@mapNotNull null
        Place(id, name.toLocaleTitle())
    }.sortedBy { it.name }
}

private fun String.toLocaleTitle(): String = lowercase(Locale("tr", "TR")).replaceFirstChar {
    if (it.isLowerCase()) it.titlecase(Locale("tr", "TR")) else it.toString()
}

private fun parsePrayerDays(json: String): List<PrayerDay> {
    val arr = JSONArray(json)
    return (0 until arr.length()).mapNotNull { i ->
        val o = arr.optJSONObject(i) ?: return@mapNotNull null
        PrayerDay(
            date = o.optString("MiladiTarihUzun", o.optString("MiladiTarihKisa")),
            hijri = o.optString("HicriTarihUzun"),
            imsak = o.optString("Imsak"), gunes = o.optString("Gunes"),
            ogle = o.optString("Ogle"), ikindi = o.optString("Ikindi"),
            aksam = o.optString("Aksam"), yatsi = o.optString("Yatsi")
        )
    }
}

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { EzanApp() }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun EzanApp() {
    val context = androidx.compose.ui.platform.LocalContext.current
    val prefs = remember { context.getSharedPreferences("ezan_prefs", 0) }
    val scope = rememberCoroutineScope()
    var cities by remember { mutableStateOf<List<Place>>(emptyList()) }
    var districts by remember { mutableStateOf<List<Place>>(emptyList()) }
    var selectedCity by remember { mutableStateOf<Place?>(null) }
    var selectedDistrict by remember { mutableStateOf<Place?>(null) }
    var days by remember { mutableStateOf<List<PrayerDay>>(emptyList()) }
    var loading by remember { mutableStateOf(false) }
    var error by remember { mutableStateOf<String?>(null) }
    var now by remember { mutableLongStateOf(System.currentTimeMillis()) }

    LaunchedEffect(Unit) {
        while (true) { now = System.currentTimeMillis(); delay(30_000) }
    }

    // Keep the loader as a local composable callback.
    val loadTimes: (Place) -> Unit = { d ->
        scope.launch {
            loading = true; error = null
            try {
                days = withContext(Dispatchers.IO) { parsePrayerDays(getJson("/vakitler/${d.id}")) }
                prefs.edit().putString("districtId", d.id).apply()
            } catch (e: Exception) { error = e.message ?: "Vakitler yüklenemedi." }
            loading = false
        }
    }

    fun loadCities() {
        scope.launch {
            loading = true; error = null
            try {
                val loaded = withContext(Dispatchers.IO) {
                    parsePlaces(getJson("/sehirler/2"), "SehirID", "SehirAdi")
                }
                cities = loaded
                val savedCity = prefs.getString("cityId", null)
                val savedDistrict = prefs.getString("districtId", null)
                val city = loaded.firstOrNull { it.id == savedCity } ?: loaded.firstOrNull { it.name.equals("Şanlıurfa", true) }
                selectedCity = city
                if (city != null) {
                    val ds = withContext(Dispatchers.IO) {
                        parsePlaces(getJson("/ilceler/${city.id}"), "IlceID", "IlceAdi")
                    }
                    districts = ds
                    val district = ds.firstOrNull { it.id == savedDistrict } ?: ds.firstOrNull { it.name.equals("Haliliye", true) } ?: ds.firstOrNull()
                    selectedDistrict = district
                    if (district != null) loadTimes(district)
                }
            } catch (e: Exception) { error = e.message ?: "Veriler alınamadı." }
            loading = false
        }
    }

    fun chooseCity(city: Place) {
        scope.launch {
            selectedCity = city; selectedDistrict = null; districts = emptyList(); days = emptyList()
            prefs.edit().putString("cityId", city.id).apply()
            loading = true; error = null
            try {
                districts = withContext(Dispatchers.IO) {
                    parsePlaces(getJson("/ilceler/${city.id}"), "IlceID", "IlceAdi")
                }
                selectedDistrict = districts.firstOrNull()
                selectedDistrict?.let { loadTimes(it) }
            } catch (e: Exception) { error = e.message ?: "İlçeler yüklenemedi." }
            loading = false
        }
    }

    fun refreshTimes() {
        selectedDistrict?.let { d -> scope.launch { loading = true; error = null
            try { days = withContext(Dispatchers.IO) { parsePrayerDays(getJson("/vakitler/${d.id}")) } }
            catch (e: Exception) { error = e.message ?: "Vakitler yüklenemedi." }
            loading = false
        } }
    }

    LaunchedEffect(Unit) { loadCities() }

    val today = remember(days, now) {
        val format = SimpleDateFormat("dd.MM.yyyy", Locale("tr", "TR")).apply { timeZone = TimeZone.getTimeZone("Europe/Istanbul") }
        val todayString = format.format(Date(now))
        days.firstOrNull { it.date.contains(todayString) || it.date == todayString } ?: days.firstOrNull()
    }
    val prayers = today?.let {
        listOf("İmsak" to it.imsak, "Güneş" to it.gunes, "Öğle" to it.ogle,
            "İkindi" to it.ikindi, "Akşam" to it.aksam, "Yatsı" to it.yatsi)
    } ?: emptyList()
    val nextPrayer = remember(prayers, now) {
        val sdf = SimpleDateFormat("HH:mm", Locale("tr", "TR"))
        val current = SimpleDateFormat("HH:mm", Locale("tr", "TR")).format(Date(now))
        prayers.firstOrNull { (_, time) -> time > current } ?: prayers.firstOrNull()
    }

    MaterialTheme(
        colorScheme = lightColorScheme(
            primary = Color(0xFF126B55), secondary = Color(0xFFB88A42),
            background = Color(0xFFF4F7F5), surface = Color.White
        )
    ) {
        Scaffold(containerColor = Color(0xFFF4F7F5)) { padding ->
            LazyColumn(
                modifier = Modifier.fillMaxSize().padding(padding),
                contentPadding = PaddingValues(18.dp),
                verticalArrangement = Arrangement.spacedBy(14.dp)
            ) {
                item {
                    Column {
                        Text("EZAN VAKİTLERİ", fontSize = 13.sp, color = Color(0xFF126B55), fontWeight = FontWeight.Bold, letterSpacing = 2.sp)
                        Text("Türkiye • 81 İl", fontSize = 28.sp, fontWeight = FontWeight.Bold, color = Color(0xFF173B32))
                        Text("Namaz vakitlerini il ve ilçene göre takip et", color = Color.Gray, fontSize = 14.sp)
                    }
                }
                item {
                    Card(shape = RoundedCornerShape(20.dp), colors = CardDefaults.cardColors(containerColor = Color(0xFF126B55))) {
                        Column(Modifier.fillMaxWidth().padding(18.dp)) {
                            Row(verticalAlignment = Alignment.CenterVertically) {
                                Icon(Icons.Default.LocationOn, null, tint = Color.White)
                                Text("${selectedCity?.name ?: "İl seç"} / ${selectedDistrict?.name ?: "İlçe seç"}",
                                    color = Color.White, fontWeight = FontWeight.SemiBold)
                                Spacer(Modifier.weight(1f))
                                IconButton(onClick = { refreshTimes() }) {
                                    Icon(Icons.Default.Refresh, "Yenile", tint = Color.White)
                                }
                            }
                            Text(today?.date ?: "Günlük namaz vakitleri", color = Color(0xFFD7E9E2), fontSize = 13.sp)
                            Spacer(Modifier.height(10.dp))
                            Text(nextPrayer?.first ?: "Vakitler", color = Color.White.copy(alpha = .85f), fontSize = 15.sp)
                            Text(nextPrayer?.second ?: "--:--", color = Color.White, fontSize = 42.sp, fontWeight = FontWeight.Bold)
                            Text("Bir sonraki vakit", color = Color(0xFFD7E9E2), fontSize = 12.sp)
                        }
                    }
                }
                item {
                    Text("Konum seçimi", fontSize = 18.sp, fontWeight = FontWeight.Bold, color = Color(0xFF173B32))
                }
                item {
                    SearchDropdown("İl seç (81 il)", cities, selectedCity) { chooseCity(it) }
                }
                item {
                    SearchDropdown("İlçe seç", districts, selectedDistrict) {
                        selectedDistrict = it
                        loadTimes(it)
                    }
                }
                item {
                    Row(verticalAlignment = Alignment.CenterVertically) {
                        Text("Bugünün vakitleri", fontSize = 18.sp, fontWeight = FontWeight.Bold, color = Color(0xFF173B32))
                        Spacer(Modifier.weight(1f))
                        TextButton(onClick = { refreshTimes() }) { Text("Yenile") }
                    }
                }
                if (loading) item { LinearProgressIndicator(Modifier.fillMaxWidth()) }
                if (error != null) item {
                    Text(error!!, color = MaterialTheme.colorScheme.error)
                }
                if (today != null) {
                    items(prayers) { (name, time) ->
                        Card(shape = RoundedCornerShape(14.dp), colors = CardDefaults.cardColors(containerColor = Color.White)) {
                            Row(Modifier.fillMaxWidth().padding(horizontal = 18.dp, vertical = 15.dp),
                                verticalAlignment = Alignment.CenterVertically) {
                                Text(name, fontSize = 16.sp, fontWeight = FontWeight.Medium, color = Color(0xFF25473D))
                                Spacer(Modifier.weight(1f))
                                Text(time.ifBlank { "--:--" }, fontSize = 20.sp, fontWeight = FontWeight.Bold, color = Color(0xFF126B55))
                            }
                        }
                    }
                    if (today.hijri.isNotBlank()) item {
                        Text("Hicrî tarih: ${today.hijri}", color = Color.Gray, fontSize = 13.sp)
                    }
                } else if (!loading && error == null) {
                    item { Text("Vakitleri görmek için il ve ilçe seçin.", color = Color.Gray) }
                }
                item {
                    Text("Veri kaynağı: Diyanet verilerine dayanan Namaz Vakti API. İnternet bağlantısı gerekir.",
                        fontSize = 11.sp, color = Color.Gray)
                }
            }
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
private fun SearchDropdown(
    label: String,
    options: List<Place>,
    selected: Place?,
    onSelect: (Place) -> Unit
) {
    var expanded by remember { mutableStateOf(false) }
    var query by remember(selected) { mutableStateOf(selected?.name ?: "") }
    val filtered = remember(options, query) {
        if (query.isBlank()) options else options.filter { it.name.contains(query, ignoreCase = true) }
    }
    ExposedDropdownMenuBox(expanded = expanded, onExpandedChange = { expanded = !expanded }) {
        OutlinedTextField(
            value = query,
            onValueChange = { query = it; expanded = true },
            modifier = Modifier.fillMaxWidth().menuAnchor(),
            label = { Text(label) },
            trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded = expanded) },
            singleLine = true,
            enabled = options.isNotEmpty()
        )
        ExposedDropdownMenu(expanded = expanded, onDismissRequest = { expanded = false }) {
            filtered.take(100).forEach { place ->
                DropdownMenuItem(
                    text = { Text(place.name) },
                    onClick = { query = place.name; expanded = false; onSelect(place) }
                )
            }
            if (filtered.isEmpty()) DropdownMenuItem(text = { Text("Sonuç bulunamadı") }, onClick = { expanded = false })
        }
    }
}name: Android APK Olustur

on:
  push:
  workflow_dispatch:

jobs:
  build-apk:
    runs-on: ubuntu-latest
    steps:
      - name: Depoyu indir
        uses: actions/checkout@v4

      - name: JDK 17 kur
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Gradle kur
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.9'

      - name: Debug APK derle
        run: |
          if [ -f "settings.gradle.kts" ]; then
            gradle assembleDebug --no-daemon
          elif [ -f "EzanVakitleriAndroid/settings.gradle.kts" ]; then
            cd EzanVakitleriAndroid && gradle assembleDebug --no-daemon
          else
            echo "settings.gradle.kts bulunamadi!"
            exit 1
          fi

      - name: APK dosyasini yukle
        uses: actions/upload-artifact@v4
        with:
          name: EzanVakitleri-debug-apk
          path: "**/app/build/outputs/apk/debug/app-debug.apk"
          if-no-files-found: error


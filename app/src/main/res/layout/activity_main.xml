package com.netdoctor

import android.Manifest
import android.app.AppOpsManager
import android.app.usage.NetworkStatsManager
import android.app.usage.UsageStatsManager
import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.net.ConnectivityManager
import android.net.NetworkCapabilities
import android.net.TrafficStats
import android.net.Uri
import android.os.Build
import android.os.Bundle
import android.os.Handler
import android.os.Looper
import android.provider.Settings
import android.telephony.TelephonyManager
import android.webkit.WebView
import android.webkit.WebViewClient
import android.widget.*
import androidx.appcompat.app.AlertDialog
import androidx.appcompat.app.AppCompatActivity
import androidx.cardview.widget.CardView
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import com.google.android.gms.ads.*
import com.google.android.gms.ads.interstitial.InterstitialAd
import com.google.android.gms.ads.interstitial.InterstitialAdLoadCallback
import kotlinx.coroutines.*
import org.json.JSONObject
import java.text.SimpleDateFormat
import java.util.*
import kotlin.collections.HashMap

class MainActivity : AppCompatActivity() {
    
    // واجهات المستخدم
    private lateinit var scrollView: ScrollView
    private lateinit var tvNetworkType: TextView
    private lateinit var tvSignalStrength: TextView
    private lateinit var tvIpAddress: TextView
    private lateinit var tvOperator: TextView
    private lateinit var tvDataUsage: TextView
    private lateinit var btnDiagnose: Button
    private lateinit var btnApnGuide: Button
    private lateinit var btnGeminiHelp: Button
    private lateinit var btnAnalytics: Button
    private lateinit var btnFirewall: Button
    private lateinit var progressBar: ProgressBar
    private lateinit var recyclerApps: RecyclerView
    private lateinit var adView: AdView
    private lateinit var geminiWebView: WebView
    
    // إعلانات
    private var mInterstitialAd: InterstitialAd? = null
    private var isProUser = false // سيتم تغييرها لاحقاً
    
    // Coroutines
    private val mainScope = CoroutineScope(Dispatchers.Main + Job())
    
    // قائمة التطبيقات للجدار الناري
    private val appsList = mutableListOf<AppInfo>()
    private lateinit var appsAdapter: AppsAdapter
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        initViews()
        setupClickListeners()
        checkPermissions()
        loadAds()
        loadInterstitialAd()
        setupRecyclerView()
        startDataCollection()
        
        // عرض توصية Gemini أسبوعياً
        checkAndShowWeeklyRecommendation()
    }
    
    private fun initViews() {
        scrollView = findViewById(R.id.scrollView)
        tvNetworkType = findViewById(R.id.tv_network_type)
        tvSignalStrength = findViewById(R.id.tv_signal_strength)
        tvIpAddress = findViewById(R.id.tv_ip_address)
        tvOperator = findViewById(R.id.tv_operator)
        tvDataUsage = findViewById(R.id.tv_data_usage)
        btnDiagnose = findViewById(R.id.btn_diagnose)
        btnApnGuide = findViewById(R.id.btn_apn_guide)
        btnGeminiHelp = findViewById(R.id.btn_gemini_help)
        btnAnalytics = findViewById(R.id.btn_analytics)
        btnFirewall = findViewById(R.id.btn_firewall)
        progressBar = findViewById(R.id.progress_bar)
        recyclerApps = findViewById(R.id.recycler_apps)
        adView = findViewById(R.id.adView)
        geminiWebView = findViewById(R.id.geminiWebView)
        
        // إخفاء WebView في البداية
        geminiWebView.visibility = android.view.View.GONE
    }
    
    private fun setupClickListeners() {
        btnDiagnose.setOnClickListener {
            diagnoseNetwork()
            showInterstitialAd()
        }
        
        btnApnGuide.setOnClickListener {
            showApnGuide()
        }
        
        btnGeminiHelp.setOnClickListener {
            showGeminiWebView()
        }
        
        btnAnalytics.setOnClickListener {
            showDetailedAnalytics()
        }
        
        btnFirewall.setOnClickListener {
            if (isProUser) {
                showFirewallDialog()
            } else {
                showProUpgradeDialog("الجدار الناري")
            }
        }
    }
    
    // ==================== 1. تشخيص الشبكة ====================
    
    private fun checkPermissions() {
        val permissions = mutableListOf<String>()
        
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.READ_PHONE_STATE)
            != PackageManager.PERMISSION_GRANTED) {
            permissions.add(Manifest.permission.READ_PHONE_STATE)
        }
        
        if (permissions.isNotEmpty()) {
            ActivityCompat.requestPermissions(this, permissions.toTypedArray(), 100)
        } else {
            diagnoseNetwork()
        }
    }
    
    private fun diagnoseNetwork() {
        progressBar.visibility = android.view.View.VISIBLE
        
        mainScope.launch {
            val networkInfo = withContext(Dispatchers.IO) {
                getNetworkInfo()
            }
            
            tvNetworkType.text = "🌐 نوع الشبكة: ${networkInfo["type"]}"
            tvSignalStrength.text = "📶 قوة الإشارة: ${networkInfo["signal"]}%"
            tvIpAddress.text = "📍 عنوان IP: ${networkInfo["ip"]}"
            tvOperator.text = "📱 المشغل: ${networkInfo["operator"]}\n📡 MCC/MNC: ${networkInfo["mcc"]}/${networkInfo["mnc"]}"
            
            progressBar.visibility = android.view.View.GONE
            
            saveDiagnosisLog(networkInfo)
            showNotification("تم تشخيص الشبكة بنجاح")
        }
    }
    
    private fun getNetworkInfo(): Map<String, String> {
        val info = mutableMapOf<String, String>()
        
        try {
            val connectivityManager = getSystemService(CONNECTIVITY_SERVICE) as ConnectivityManager
            val activeNetwork = connectivityManager.activeNetwork
            val capabilities = connectivityManager.getNetworkCapabilities(activeNetwork)
            
            info["type"] = when {
                capabilities?.hasTransport(NetworkCapabilities.TRANSPORT_WIFI) == true -> "Wi-Fi 📶"
                capabilities?.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR) == true -> {
                    val tm = getSystemService(TELEPHONY_SERVICE) as TelephonyManager
                    when (tm.dataNetworkType) {
                        TelephonyManager.NETWORK_TYPE_LTE -> "4G/LTE 🚀"
                        TelephonyManager.NETWORK_TYPE_NR -> "5G ⚡"
                        TelephonyManager.NETWORK_TYPE_HSPAP -> "3G/HSPA+ 📱"
                        else -> "محمول 📱"
                    }
                }
                else -> "غير متصل ❌"
            }
            
            info["signal"] = if (info["type"] == "Wi-Fi 📶") {
                val wifiManager = applicationContext.getSystemService(WIFI_SERVICE) as android.net.wifi.WifiManager
                val wifiInfo = wifiManager.connectionInfo
                val rssi = wifiInfo.rssi
                ((rssi + 100) * 100 / 55).coerceIn(0, 100).toString()
            } else {
                getCellularSignalStrength().toString()
            }
            
            info["ip"] = getLocalIpAddress()
            
            val tm = getSystemService(TELEPHONY_SERVICE) as TelephonyManager
            if (ActivityCompat.checkSelfPermission(this, Manifest.permission.READ_PHONE_STATE)
                == PackageManager.PERMISSION_GRANTED) {
                info["operator"] = tm.networkOperatorName ?: "غير معروف"
                val networkOperator = tm.networkOperator ?: ""
                info["mcc"] = if (networkOperator.length >= 3) networkOperator.substring(0, 3) else "000"
                info["mnc"] = if (networkOperator.length >= 5) networkOperator.substring(3, 5) else "00"
            } else {
                info["operator"] = "غير معروف"
                info["mcc"] = "000"
                info["mnc"] = "00"
            }
            
        } catch (e: Exception) {
            info["type"] = "خطأ: ${e.message}"
            info["signal"] = "0"
            info["ip"] = "غير متوفر"
            info["operator"] = "غير معروف"
            info["mcc"] = "000"
            info["mnc"] = "00"
        }
        
        return info
    }
    
    private fun getCellularSignalStrength(): Int {
        return try {
            val tm = getSystemService(TELEPHONY_SERVICE) as TelephonyManager
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                tm.signalStrength?.level?.let { (it * 100 / 4) } ?: 50
            } else {
                75 // قيمة افتراضية
            }
        } catch (e: Exception) {
            50
        }
    }
    
    private fun getLocalIpAddress(): String {
        return try {
            java.net.NetworkInterface.getNetworkInterfaces()?.toList()
                ?.flatMap { it.inetAddresses.toList() }
                ?.firstOrNull { !it.isLoopbackAddress && it.hostAddress?.contains(':') != true }
                ?.hostAddress ?: "غير متوفر"
        } catch (e: Exception) {
            "غير متوفر"
        }
    }
    
    // ==================== 2. دليل APN العالمي ====================
    
    private fun showApnGuide() {
        val countries = arrayOf(
            "🇸🇦 السعودية",
            "🇪🇬 مصر",
            "🇦🇪 الإمارات",
            "🇰🇼 الكويت",
            "🇶🇦 قطر",
            "🇧🇭 البحرين",
            "🇴🇲 عمان",
            "🇯🇴 الأردن",
            "🇱🇧 لبنان",
            "🇾🇪 اليمن",
            "🇸🇩 السودان",
            "🇩🇿 الجزائر",
            "🇲🇦 المغرب",
            "🇹🇳 تونس",
            "🇱🇾 ليبيا",
            "🇮🇶 العراق",
            "🇵🇸 فلسطين"
        )
        
        AlertDialog.Builder(this)
            .setTitle("📖 دليل إعدادات APN - اختر الدولة")
            .setItems(countries) { _, which ->
                when (which) {
                    0 -> showSaudiOperators()
                    1 -> showEgyptOperators()
                    2 -> showUaeOperators()
                    3 -> showKuwaitOperators()
                    4 -> showQatarOperators()
                    5 -> showBahrainOperators()
                    6 -> showOmanOperators()
                    7 -> showJordanOperators()
                    8 -> showLebanonOperators()
                    9 -> showYemenOperators()
                    10 -> showSudanOperators()
                    11 -> showAlgeriaOperators()
                    12 -> showMoroccoOperators()
                    13 -> showTunisiaOperators()
                    14 -> showLibyaOperators()
                    15 -> showIraqOperators()
                    16 -> showPalestineOperators()
                }
            }
            .show()
    }
    
    private fun showSaudiOperators() {
        val operators = arrayOf("STC", "Mobily", "Zain", "Lebara", "Virgin")
        AlertDialog.Builder(this)
            .setTitle("🇸🇦 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("STC Internet", "stc", "default,supl", "420", "01")
                    1 -> showApnDetails("Mobily Internet", "mobily", "default,supl", "420", "03")
                    2 -> showApnDetails("Zain Internet", "zain", "default,supl", "420", "04")
                    3 -> showApnDetails("Lebara Internet", "lebara", "default", "420", "05")
                    4 -> showApnDetails("Virgin Internet", "virgin", "default", "420", "06")
                }
            }
            .show()
    }
    
    private fun showEgyptOperators() {
        val operators = arrayOf("Vodafone", "Orange", "Etisalat", "WE")
        AlertDialog.Builder(this)
            .setTitle("🇪🇬 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Vodafone Internet", "vodafone", "default", "602", "01")
                    1 -> showApnDetails("Orange Internet", "orange", "default", "602", "02")
                    2 -> showApnDetails("Etisalat Internet", "etisalat", "default", "602", "03")
                    3 -> showApnDetails("WE Internet", "we", "default", "602", "04")
                }
            }
            .show()
    }
    
    private fun showUaeOperators() {
        val operators = arrayOf("Etisalat", "Du")
        AlertDialog.Builder(this)
            .setTitle("🇦🇪 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Etisalat Internet", "etisalat", "default", "424", "03")
                    1 -> showApnDetails("Du Internet", "du", "default", "424", "02")
                }
            }
            .show()
    }
    
    private fun showKuwaitOperators() {
        val operators = arrayOf("Zain", "Ooredoo", "STC")
        AlertDialog.Builder(this)
            .setTitle("🇰🇼 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Zain Internet", "zain", "default", "419", "02")
                    1 -> showApnDetails("Ooredoo Internet", "ooredoo", "default", "419", "03")
                    2 -> showApnDetails("STC Internet", "stc", "default", "419", "04")
                }
            }
            .show()
    }
    
    private fun showQatarOperators() {
        val operators = arrayOf("Ooredoo", "Vodafone")
        AlertDialog.Builder(this)
            .setTitle("🇶🇦 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Ooredoo Internet", "ooredoo", "default", "427", "01")
                    1 -> showApnDetails("Vodafone Internet", "vodafone", "default", "427", "02")
                }
            }
            .show()
    }
    
    private fun showBahrainOperators() {
        val operators = arrayOf("Batelco", "Viva", "Zain")
        AlertDialog.Builder(this)
            .setTitle("🇧🇭 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Batelco Internet", "batelco", "default", "426", "01")
                    1 -> showApnDetails("Viva Internet", "viva", "default", "426", "02")
                    2 -> showApnDetails("Zain Internet", "zain", "default", "426", "03")
                }
            }
            .show()
    }
    
    private fun showOmanOperators() {
        val operators = arrayOf("Omantel", "Ooredoo", "Vodafone")
        AlertDialog.Builder(this)
            .setTitle("🇴🇲 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Omantel Internet", "omantel", "default", "422", "02")
                    1 -> showApnDetails("Ooredoo Internet", "ooredoo", "default", "422", "03")
                    2 -> showApnDetails("Vodafone Internet", "vodafone", "default", "422", "04")
                }
            }
            .show()
    }
    
    private fun showJordanOperators() {
        val operators = arrayOf("Zain", "Orange", "Umniah")
        AlertDialog.Builder(this)
            .setTitle("🇯🇴 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Zain Internet", "zain", "default", "416", "01")
                    1 -> showApnDetails("Orange Internet", "orange", "default", "416", "02")
                    2 -> showApnDetails("Umniah Internet", "umniah", "default", "416", "03")
                }
            }
            .show()
    }
    
    private fun showLebanonOperators() {
        val operators = arrayOf("Touch", "Alfa")
        AlertDialog.Builder(this)
            .setTitle("🇱🇧 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Touch Internet", "touch", "default", "415", "01")
                    1 -> showApnDetails("Alfa Internet", "alfa", "default", "415", "03")
                }
            }
            .show()
    }
    
    private fun showYemenOperators() {
        val operators = arrayOf("Yemen Mobile", "MTN", "You", "Sabafon")
        AlertDialog.Builder(this)
            .setTitle("🇾🇪 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Yemen Mobile Internet", "yemenmobile", "default", "421", "04")
                    1 -> showApnDetails("MTN Internet", "mtn", "default", "421", "01")
                    2 -> showApnDetails("You Internet", "you", "default", "421", "02")
                    3 -> showApnDetails("Sabafon Internet", "sabafon", "default", "421", "03")
                }
            }
            .show()
    }
    
    private fun showSudanOperators() {
        val operators = arrayOf("Sudani", "Zain", "MTN")
        AlertDialog.Builder(this)
            .setTitle("🇸🇩 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Sudani Internet", "sudani", "default", "634", "01")
                    1 -> showApnDetails("Zain Internet", "zain", "default", "634", "02")
                    2 -> showApnDetails("MTN Internet", "mtn", "default", "634", "03")
                }
            }
            .show()
    }
    
    private fun showAlgeriaOperators() {
        val operators = arrayOf("Mobilis", "Djezzy", "Ooredoo")
        AlertDialog.Builder(this)
            .setTitle("🇩🇿 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Mobilis Internet", "mobilis", "default", "603", "01")
                    1 -> showApnDetails("Djezzy Internet", "djezzy", "default", "603", "02")
                    2 -> showApnDetails("Ooredoo Internet", "ooredoo", "default", "603", "03")
                }
            }
            .show()
    }
    
    private fun showMoroccoOperators() {
        val operators = arrayOf("Maroc Telecom", "Orange", "Inwi")
        AlertDialog.Builder(this)
            .setTitle("🇲🇦 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Maroc Telecom", "mt", "default", "604", "01")
                    1 -> showApnDetails("Orange Internet", "orange", "default", "604", "02")
                    2 -> showApnDetails("Inwi Internet", "inwi", "default", "604", "03")
                }
            }
            .show()
    }
    
    private fun showTunisiaOperators() {
        val operators = arrayOf("Tunisie Telecom", "Orange", "Ooredoo")
        AlertDialog.Builder(this)
            .setTitle("🇹🇳 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Tunisie Telecom", "tt", "default", "605", "01")
                    1 -> showApnDetails("Orange Internet", "orange", "default", "605", "02")
                    2 -> showApnDetails("Ooredoo Internet", "ooredoo", "default", "605", "03")
                }
            }
            .show()
    }
    
    private fun showLibyaOperators() {
        val operators = arrayOf("Libyana", "AlMadar")
        AlertDialog.Builder(this)
            .setTitle("🇱🇾 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Libyana Internet", "libyana", "default", "606", "01")
                    1 -> showApnDetails("AlMadar Internet", "almadar", "default", "606", "02")
                }
            }
            .show()
    }
    
    private fun showIraqOperators() {
        val operators = arrayOf("Zain", "AsiaCell", "Korek")
        AlertDialog.Builder(this)
            .setTitle("🇮🇶 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Zain Internet", "zain", "default", "418", "01")
                    1 -> showApnDetails("AsiaCell Internet", "asiacell", "default", "418", "02")
                    2 -> showApnDetails("Korek Internet", "korek", "default", "418", "03")
                }
            }
            .show()
    }
    
    private fun showPalestineOperators() {
        val operators = arrayOf("Jawwal", "Ooredoo")
        AlertDialog.Builder(this)
            .setTitle("🇵🇸 اختر المشغل")
            .setItems(operators) { _, which ->
                when (which) {
                    0 -> showApnDetails("Jawwal Internet", "jawwal", "default", "425", "01")
                    1 -> showApnDetails("Ooredoo Internet", "ooredoo", "default", "425", "02")
                }
            }
            .show()
    }
    
    private fun showApnDetails(name: String, apn: String, apnType: String, mcc: String, mnc: String) {
        val message = """
            📱 إعدادات APN المقترحة:
            
            📛 الاسم: $name
            🔗 APN: $apn
            📡 نوع APN: $apnType
            🌍 MCC: $mcc
            📍 MNC: $mnc
            🔐 البروتوكول: IPv4/IPv6
            
            💡 نصيحة: بعد إضافة الإعدادات، اضغط على حفظ ثم أعد تشغيل الهاتف
        """.trimIndent()
        
        AlertDialog.Builder(this)
            .setTitle("⚙️ إعدادات APN")
            .setMessage(message)
            .setPositiveButton("فتح إعدادات APN") { _, _ ->
                startActivity(Intent(Settings.ACTION_APN_SETTINGS))
            }
            .setNegativeButton("إلغاء", null)
            .show()
    }
    
    // ==================== 3. مساعدة Gemini (بدون مفتاح API) ====================
    
    private fun showGeminiWebView() {
        // إظهار WebView بدلاً من الحوار
        geminiWebView.visibility = android.view.View.VISIBLE
        scrollView.visibility = android.view.View.GONE
        
        // تهيئة WebView
        geminiWebView.settings.javaScriptEnabled = true
        geminiWebView.settings.domStorageEnabled = true
        geminiWebView.webViewClient = object : WebViewClient() {
            override fun onPageFinished(view: WebView?, url: String?) {
                progressBar.visibility = android.view.View.GONE
                // حقن برومبت مسبق لمساعدة المستخدم
                if (url?.contains("gemini.google.com") == true) {
                    Handler(Looper.getMainLooper()).postDelayed({
                        injectPrompt()
                    }, 3000)
                }
            }
        }
        
        // فتح Gemini Web المجاني
        geminiWebView.loadUrl("https://gemini.google.com")
        progressBar.visibility = android.view.View.VISIBLE
        
        // زر العودة
        Toast.makeText(this, "💡 اكتب مشكلتك في الدردشة مع Gemini", Toast.LENGTH_LONG).show()
    }
    
    private fun injectPrompt() {
        val script = """
            javascript: (function() {
                var textarea = document.querySelector('textarea');
                if (textarea) {
                    textarea.value = 'أنا خبير شبكات. ساعدني في إعدادات APN لمشغلي في اليمن (يمن موبايل). الإنترنت بطيء جداً.';
                    textarea.dispatchEvent(new Event('input', { bubbles: true }));
                    var button = document.querySelector('button[aria-label="Send"]');
                    if (button) button.click();
                }
            })();
        """.trimIndent()
        
        geminiWebView.evaluateJavascript(script, null)
    }
    
    // زر العودة من WebView
    override fun onBackPressed() {
        if (geminiWebView.visibility == android.view.View.VISIBLE) {
            geminiWebView.visibility = android.view.View.GONE
            scrollView.visibility = android.view.View.VISIBLE
        } else {
            super.onBackPressed()
        }
    }
    
    // ==================== 4. تحليلات الاستهلاك ====================
    
    private fun setupRecyclerView() {
        recyclerApps.layoutManager = LinearLayoutManager(this)
        appsAdapter = AppsAdapter(appsList) { appInfo ->
            if (isProUser) {
                showAppDataDetails(appInfo)
            } else {
                showProUpgradeDialog("تفاصيل الاستهلاك المتقدمة")
            }
        }
        recyclerApps.adapter = appsAdapter
    }
    
    private fun startDataCollection() {
        mainScope.launch {
            while (true) {
                collectDataUsage()
                delay(60000) // كل دقيقة
            }
        }
    }
    
    private fun collectDataUsage() {
        mainScope.launch {
            val usage = withContext(Dispatchers.IO) {
                getTopAppsDataUsage()
            }
            
            // تحديث الواجهة
            val totalMB = usage.sumOf { it.mobileBytes + it.wifiBytes } / (1024 * 1024)
            tvDataUsage.text = "📊 إجمالي الاستهلاك اليوم: ${totalMB} MB"
            
            // تحديث القائمة
            appsList.clear()
            appsList.addAll(usage.take(10))
            appsAdapter.notifyDataSetChanged()
        }
    }
    
    private fun getTopAppsDataUsage(): List<AppInfo> {
        val usageMap = HashMap<String, AppInfo>()
        
        try {
            val usageStatsManager = getSystemService(USAGE_STATS_SERVICE) as UsageStatsManager
            val endTime = System.currentTimeMillis()
            val startTime = endTime - 24 * 60 * 60 * 1000 // آخر 24 ساعة
            
            val stats = usageStatsManager.queryUsageStats(
                UsageStatsManager.INTERVAL_DAILY,
                startTime,
                endTime
            )
            
            val pm = packageManager
            
            for (stat in stats) {
                try {
                    val appInfo = pm.getApplicationInfo(stat.packageName, 0)
                    val appName = pm.getApplicationLabel(appInfo).toString()
                    
                    // تقدير استهلاك البيانات (تقريبي)
                    val mobileBytes = stat.totalTimeInForeground * 1000 // تقريب
                    val wifiBytes = stat.totalTimeVisible * 500 // تقريب
                    
                    usageMap[stat.packageName] = AppInfo(
                        packageName = stat.packageName,
                        appName = appName,
                        mobileBytes = mobileBytes,
                        wifiBytes = wifiBytes,
                        foregroundTime = stat.totalTimeInForeground,
                        backgroundTime = stat.totalTimeVisible
                    )
                } catch (e: Exception) {
                    // تجاهل التطبيقات بدون اسم
                }
            }
        } catch (e: Exception) {
            e.printStackTrace()
        }
        
        return usageMap.values.sortedByDescending { it.mobileBytes + it.wifiBytes }
    }
    
    private fun showDetailedAnalytics() {
        if (!hasUsageStatsPermission()) {
            requestUsageStatsPermission()
            return
        }
        
        val totalUsage = appsList.sumOf { it.mobileBytes + it.wifiBytes } / (1024 * 1024)
        val topApp = appsList.firstOrNull()
        
        val message = """
            📊 تقرير استهلاك البيانات:
            
            📦 إجمالي اليوم: $totalUsage MB
            
            🥇 أعلى تطبيق استهلاكاً:
            ${topApp?.appName}: ${(topApp?.mobileBytes ?: 0 + (topApp?.wifiBytes ?: 0)) / (1024 * 1024)} MB
            
            📱 عدد التطبيقات المستخدمة: ${appsList.size}
            
            💡 نصيحة: راجع إعدادات كل تطبيق للحد من استهلاك الخلفية
        """.trimIndent()
        
        AlertDialog.Builder(this)
            .setTitle("📈 تحليلات متقدمة")
            .setMessage(message)
            .setPositiveButton("فتح الإعدادات") { _, _ ->
                startActivity(Intent(Settings.ACTION_USAGE_ACCESS_SETTINGS))
            }
            .setNegativeButton("إغلاق", null)
            .show()
    }
    
    private fun hasUsageStatsPermission(): Boolean {
        val appOps = getSystemService(APP_OPS_SERVICE) as AppOpsManager
        val mode = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            appOps.unsafeCheckOpNoThrow(
                AppOpsManager.OPSTR_GET_USAGE_STATS,
                android.os.Process.myUid(),
                packageName
            )
        } else {
            @Suppress("DEPRECATION")
            appOps.checkOpNoThrow(
                AppOpsManager.OPSTR_GET_USAGE_STATS,
                android.os.Process.myUid(),
                packageName
            )
        }
        return mode == AppOpsManager.MODE_ALLOWED
    }
    
    private fun requestUsageStatsPermission() {
        AlertDialog.Builder(this)
            .setTitle("⚠️ صلاحية مطلوبة")
            .setMessage("لتحليل استهلاك البيانات، نحتاج إلى صلاحية الوصول لإحصائيات الاستخدام")
            .setPositiveButton("منح الصلاحية") { _, _ ->
                startActivity(Intent(Settings.ACTION_USAGE_ACCESS_SETTINGS))
            }
            .setNegativeButton("تذكر لاحقاً", null)
            .show()
    }
    
    private fun showAppDataDetails(appInfo: AppInfo) {
        val totalMB = (appInfo.mobileBytes + appInfo.wifiBytes) / (1024 * 1024)
        val mobileMB = appInfo.mobileBytes / (1024 * 1024)
        val wifiMB = appInfo.wifiBytes / (1024 * 1024)
        
        val message = """
            📱 التطبيق: ${appInfo.appName}
            
            📊 إجمالي الاستهلاك: $totalMB MB
            📡 بيانات الجوال: $mobileMB MB
            📶 بيانات الواي فاي: $wifiMB MB
            
            ⏱️ وقت المقدمة: ${appInfo.foregroundTime / 60000} دقيقة
            ⏰ وقت الخلفية: ${appInfo.backgroundTime / 60000} دقيقة
        """.trimIndent()
        
        AlertDialog.Builder(this)
            .setTitle("تفاصيل الاستهلاك")
            .setMessage(message)
            .setPositiveButton("تقييد بيانات الخلفية") { _, _ ->
                try {
                    val intent = Intent(Settings.ACTION_IGNORE_BACKGROUND_DATA_RESTRICTIONS_SETTINGS)
                    intent.data = Uri.parse("package:${appInfo.packageName}")
                    startActivity(intent)
                } catch (e: Exception) {
                    startActivity(Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS)
                        .setData(Uri.parse("package:${appInfo.packageName}")))
                }
            }
            .setNegativeButton("إلغاء", null)
            .show()
    }
    
    // ==================== 5. الجدار الناري (TODO للإصدارات المستقبلية) ====================
    
    private fun showFirewallDialog() {
        AlertDialog.Builder(this)
            .setTitle("🔥 الجدار الناري (قيد التطوير)")
            .setMessage("""
                ستسمح لك هذه الميزة ب:
                ✓ منع تطبيقات معينة من استخدام الإنترنت تماماً
                ✓ التحكم في الوصول للواي فاي والبيانات الخلوية بشكل منفصل
                ✓ توفير البيانات ومنع التسريبات الخلفية
                
                سيتم إطلاق هذه الميزة في الإصدار 2.0 القادم!
            """.trimIndent())
            .setPositiveButton("متابعة", null)
            .show()
    }
    
    // ==================== 6. الإعلانات والترويج للنسخة المدفوعة ====================
    
    private fun loadAds() {
        if (!isProUser) {
            MobileAds.initialize(this) {}
            val adRequest = AdRequest.Builder().build()
            adView.loadAd(adRequest)
        }
    }
    
    private fun loadInterstitialAd() {
        if (!isProUser) {
            InterstitialAd.load(this, "ca-app-pub-3940256099942544/1033173712", AdRequest.Builder().build(),
                object : InterstitialAdLoadCallback() {
                    override fun onAdLoaded(ad: InterstitialAd) {
                        mInterstitialAd = ad
                    }
                    override fun onAdFailedToLoad(adError: LoadAdError) {
                        mInterstitialAd = null
                    }
                })
        }
    }
    
    private fun showInterstitialAd() {
        if (!isProUser && mInterstitialAd != null) {
            mInterstitialAd?.show(this)
            loadInterstitialAd() // تحميل الإعلان التالي
        }
    }
    
    private fun showProUpgradeDialog(featureName: String) {
        AlertDialog.Builder(this)
            .setTitle("✨ ميزة Pro: $featureName")
            .setMessage("""
                هذه الميزة متاحة فقط في النسخة المدفوعة.
                
                🚀 مميزات نسخة Pro:
                • إزالة جميع الإعلانات
                • الجدار الناري (منع تطبيقات من الإنترنت)
                • تحليلات متقدمة ورسوم بيانية
                • توصيات غير محدودة من Gemini
                • دعم الأولوية
                
                السعر: 4.99 دولار فقط (دفعة واحدة)
            """.trimIndent())
            .setPositiveButton("الترقية الآن") { _, _ ->
                // TODO: إضافة Google Play Billing
                Toast.makeText(this, "جاري تطوير نظام الدفع... سيتم إطلاقه قريباً", Toast.LENGTH_LONG).show()
            }
            .setNegativeButton("تذكر لاحقاً", null)
            .show()
    }
    
    // ==================== 7. التوصيات الأسبوعية ====================
    
    private fun checkAndShowWeeklyRecommendation() {
        val prefs = getSharedPreferences("netdoctor", MODE_PRIVATE)
        val lastRecommendation = prefs.getLong("last_recommendation", 0)
        val now = System.currentTimeMillis()
        val oneWeek = 7 * 24 * 60 * 60 * 1000L
        
        if (now - lastRecommendation > oneWeek) {
            showRecommendation()
            prefs.edit().putLong("last_recommendation", now).apply()
        }
    }
    
    private fun showRecommendation() {
        val topApp = appsList.firstOrNull()
        val recommendation = if (topApp != null && (topApp.backgroundTime > 30 * 60 * 1000)) {
            "📱 تطبيق ${topApp.appName} يعمل في الخلفية لأكثر من ${topApp.backgroundTime / 60000} دقيقة. قم بتقييد بيانات الخلفية لتوفير البيانات والبطارية."
        } else {
            "💡 نصيحة: استخدم الواي فاي كلما أمكن، وقم بتعطيل التحديث التلقائي للتطبيقات عبر البيانات الخلوية."
        }
        
        AlertDialog.Builder(this)
            .setTitle("💡 توصية الأسبوع")
            .setMessage(recommendation)
            .setPositiveButton("تطبيق التوصية") { _, _ ->
                if (topApp != null) {
                    startActivity(Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS)
                        .setData(Uri.parse("package:${topApp.packageName}")))
                }
            }
            .setNegativeButton("تذكر لاحقاً", null)
            .show()
    }
    
    // ==================== 8. دوال مساعدة ====================
    
    private fun saveDiagnosisLog(info: Map<String, String>) {
        try {
            val file = java.io.File(filesDir, "diagnosis_log.txt")
            val timestamp = SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault()).format(Date())
            file.appendText("$timestamp: $info\n")
            
            // الاحتفاظ بآخر 100 سجل فقط
            val lines = file.readLines()
            if (lines.size > 100) {
                file.writeText(lines.takeLast(100).joinToString("\n"))
            }
        } catch (e: Exception) {
            // تجاهل أخطاء الكتابة
        }
    }
    
    private fun showNotification(message: String) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
    }
    
    override fun onDestroy() {
        super.onDestroy()
        mainScope.cancel()
    }
}

// ==================== نماذج البيانات ====================

data class AppInfo(
    val packageName: String,
    val appName: String,
    val mobileBytes: Long,
    val wifiBytes: Long,
    val foregroundTime: Long,
    val backgroundTime: Long
)

// ==================== محول RecyclerView ====================

class AppsAdapter(
    private val appsList: List<AppInfo>,
    private val onItemClick: (AppInfo) -> Unit
) : RecyclerView.Adapter<AppsAdapter.ViewHolder>() {
    
    class ViewHolder(itemView: android.view.View) : RecyclerView.ViewHolder(itemView) {
        val tvAppName: TextView = itemView.findViewById(android.R.id.text1)
        val tvUsage: TextView = itemView.findViewById(android.R.id.text2)
    }
    
    override fun onCreateViewHolder(parent: android.view.ViewGroup, viewType: Int): ViewHolder {
        val view = android.view.LayoutInflater.from(parent.context)
            .inflate(android.R.layout.simple_list_item_2, parent, false)
        return ViewHolder(view)
    }
    
    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        val app = appsList[position]
        val totalMB = (app.mobileBytes + app.wifiBytes) / (1024 * 1024)
        holder.tvAppName.text = app.appName
        holder.tvUsage.text = "${totalMB} MB"
        holder.itemView.setOnClickListener { onItemClick(app) }
    }
    
    override fun getItemCount(): Int = appsList.size
}

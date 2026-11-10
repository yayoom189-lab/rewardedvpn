class MainActivity : AppCompatActivity() {

    private var rewardedAd: RewardedAd? = null
    private var countDown: CountDownTimer? = null
    private lateinit var tvTimer: TextView
    private lateinit var btnWatchAd: Button
    private val prefs by lazy { getSharedPreferences("vpn", MODE_PRIVATE) }

    companion object {
        const val AD_UNIT = "ca-app-pub-3940256099942544/5224354917" // TEST
        const val ONE_HOUR = 60 * 60 * 1000L
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        tvTimer = findViewById(R.id.tvTimer)
        btnWatchAd = findViewById(R.id.btnWatchAd)

        MobileAds.initialize(this) {}
        loadAd()

        btnWatchAd.setOnClickListener { showAd() }
        startTimerIfActive()
    }

    private fun loadAd() {
        btnWatchAd.isEnabled = false
        RewardedAd.load(this, AD_UNIT, AdRequest.Builder().build(),
            object : RewardedAdLoadCallback() {
                override fun onAdLoaded(ad: RewardedAd) {
                    rewardedAd = ad
                    btnWatchAd.isEnabled = true
                }
                override fun onAdFailedToLoad(e: LoadAdError) {
                    rewardedAd = null
                    Toast.makeText(this@MainActivity,
                        "Ad load failed: ${e.message}", Toast.LENGTH_SHORT).show()
                }
            })
    }

    private fun showAd() {
        val ad = rewardedAd ?: return
        ad.fullScreenContentCallback = object : FullScreenContentCallback() {
            override fun onAdDismissedFullScreenContent() {
                rewardedAd = null
                loadAd() // နောက်တစ်ကြိမ်အတွက် ကြိုတင် load
            }
        }
        ad.show(this) { _ -> addOneHour() } // ad အဆုံးထိ ကြည့်မှ ဒီနေရာရောက်
    }

    private fun addOneHour() {
        val now = System.currentTimeMillis()
        val base = maxOf(prefs.getLong("expiry", 0L), now)
        prefs.edit().putLong("expiry", base + ONE_HOUR).apply()
        startTimerIfActive()
        // TODO: ဒီနေရာမှာ VPN service စတင်မယ်
    }

    private fun startTimerIfActive() {
        countDown?.cancel()
        val remain = prefs.getLong("expiry", 0L) - System.currentTimeMillis()
        if (remain <= 0) { tvTimer.text = "00:00:00"; return }

        countDown = object : CountDownTimer(remain, 1000) {
            override fun onTick(ms: Long) {
                val s = ms / 1000
                tvTimer.text = "%02d:%02d:%02d".format(s / 3600, (s % 3600) / 60, s % 60)
            }
            override fun onFinish() {
                tvTimer.text = "00:00:00"
                // TODO: ဒီနေရာမှာ VPN service ရပ်မယ်
            }
        }.start()
    }

    override fun onDestroy() { countDown?.cancel(); super.onDestroy() }
}

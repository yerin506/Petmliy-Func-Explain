# Petmliy-Func-Explain

# 3. 기능 구현
##   각 액티비티 기능 설명
|  클래스              | 기능                     |layout                         |
|----------------|-------------------------------|-----------------------------|
|Home            |  구글 로그인, 날씨              |fragment_home.xml               |
|Analysis        |동물 감정 분석                   |fragment_analysis.xml           |
|		         |동물 감정 분석 결과              |fragment_result.xml           |
|BookMark        |즐겨 찾기 장소 보기               |fragment_likeplace.xml       |
|Post            |게시물 보기                       |fragment_post.xml             |
|                |게시물 업로드                    |fragment_upload.xml             |
|                |좋아요 한 게시물 보기             |fragment_post_like.xml       |
|                |댓글 보기                        |fragment_comment.xml       |
|Walk            |날짜별 산책 기록 보기             |fragment_walk.xml           |
|                |산책 기록 자세히 보기             |fragment_detail_tracking.xml     |
|                |산책 시작 후 트래킹 모드           |fragment_tracking.xml        |
|Map             |장소 검색                        |fragment_map.xml   |

## 로딩 화면

어플이 실행 되면 가장 먼저 보여지는 화면이다.

(로딩 화면 추가)

#### splashActivity.kt

Handler 를 이용해 2초의 딜레이 진행 뒤에야 MainActivity 화면으로 전환 된다.

```kotlin
Handler(Looper.getMainLooper()).postDelayed({  
  val intent =  
        Intent(baseContext, MainActivity::class.java)  
    startActivity (intent)  
    finish()  
    }, 2000)
```

## 메인 화면
#### MainActivity.kt

BottomNavigationView를 이용해 하단 탭으로 화면을 이동한다.
총 4개의 화면으로 구성되어 있으며 초기 화면은 HomeFragment으로 설정되어 있다.

```kotlin
bottomNavigationView.setOnItemSelectedListener { item ->  
  val account = GoogleSignIn.getLastSignedInAccount(this)  
    if(account != null) {  
        when (item.itemId) {  
  
            R.id.home -> navController.navigate(R.id.homeFragment)  
            R.id.story -> navController.navigate(R.id.postFragment)  
            R.id.walk -> navController.navigate(R.id.walkFragment)  
            R.id.map -> navController.navigate(R.id.mapFragment)  
        }  
        true  
  } else {  
        Toast.makeText(this,"로그인후 이용해주세요",Toast.LENGTH_SHORT).show()  
        true  
  }  
}
```

## 홈 화면
#### HomeFragment.kt
(홈 화면 추가)

### 로그인
로그인 방법은 구글 로그인이다.
만약 로그인이 되어 있지 않다면 버튼을 누르더라도 홈 화면이 아닌 다른 화면으로 넘어가지 못한다.

#### 구글 로그인

```kotlin
 private fun handleSignInResult(completedTask: Task<GoogleSignInAccount>) {  
        try {  
            val account = completedTask.getResult(ApiException::class.java)  
  updateUI(account)  
        } catch (e: ApiException) {  
            Log.w("Google", "signInResult:failed code=" + e.statusCode)  
            updateUI(null)  
        }  
    }
    
    private fun googleSet() {  
    val gso = GoogleSignInOptions.Builder(GoogleSignInOptions.DEFAULT_SIGN_IN)  
        .requestEmail()  
        .build()  
  
    mGoogleSignInClient = GoogleSignIn.getClient(requireActivity(), gso)  
}
```
### 날씨
OpenWeatherMap API에서 Retrofit을 이용해 날씨 상태를 받아와 실시간으로 보여준다.
현재 온도, 바람, 구름, 습도를 확인할 수 있다.

#### WeatherAPIClient.kt
```kotlin
object WeatherAPIClient {  
    fun getClient(url: String): Retrofit {  
        val okHttpClient: OkHttpClient = OkHttpClient.Builder().addInterceptor(CookiesIntercepter())  
            .addNetworkInterceptor(CookiesIntercepter()).build()  
  
        val retrofit: Retrofit = Retrofit.Builder().baseUrl(url)  
            .addConverterFactory(GsonConverterFactory.create())  
            .client(okHttpClient)  
            .build()  
  
        return retrofit  
    }  
}
```
#### WeatherAPIService.kt
```kotlin
@GET("data/2.5/{path}")  
fun doGetJsonDataWeather(  
    @Path("path") path: String,  
  @Query("q") q: String,  
  @Query("appid") appid: String,  
): Call<WeatherModel>
```

#### RemoteDataSourceImpl.kt
```kotlin
override fun getWeatherInfo(  
    jsonObject: JSONObject,  
  onResponse: (Response<WeatherModel>) -> Unit,  
  onFailure: (Throwable) -> Unit  
) {  
    val APIService: WeatherAPIService = WeatherAPIClient.getClient(  
        jsonObject.get("url").toString()  
    ).create(WeatherAPIService::class.java)  
    APIService.doGetJsonDataWeather(  
        jsonObject.get("path").toString(),  
  jsonObject.get("q").toString(),  
  jsonObject.get("appid").toString()  
    ).enqueue(object :  
        Callback<WeatherModel> {  
        override fun onResponse(call: Call<WeatherModel>, response: Response<WeatherModel>) {  
            onResponse(response)  
        }  
        override fun onFailure(call: Call<WeatherModel>, t: Throwable) {  
            onFailure(t)  
        }  
    }  
    )  
}
```

#### HomeFragment.kt에 날씨 띄우기

```kotlin
private fun initWeatherView() {  
    viewModel = ViewModelProvider(this).get(WeatherViewModel::class.java)  
    var jsonObject = JSONObject()  
    jsonObject.put("url", getString(R.string.weather_url))  
    jsonObject.put("path", "weather")  
    jsonObject.put("q", "Seoul")  
    jsonObject.put("appid", getString(R.string.weather_app_id))  
    viewModel.getWeatherInfoView(jsonObject)  
}

private fun observeData() {  
    viewModel.isSuccWeather.observe(  
        viewLifecycleOwner, Observer { it ->  
  if (it) {  
                viewModel.responseWeather.observe(  
                    viewLifecycleOwner, Observer { setWeatherData(it) }  
  )  
            }  
        }  
  )  
}  
  
private fun setWeatherData(model: WeatherModel) {  
    val temp = model.main.temp!!.toDouble() - 273.15  
  val weatherImgUrl = "http://openweathermap.org/img/w/" + model.weather[0].icon + ".png"  
  binding.currentTemp.text =  
        StringBuilder().append(String.format("%.2f", temp)).append(" 'C").toString()  
    binding.currentMain.text = model.weather[0].main  
 binding.windSpeed.text = StringBuilder().append(model.wind.speed).append(" m/s").toString()  
    binding.cloudCover.text = StringBuilder().append(model.clouds.all).append(" %").toString()  
    binding.humidity.text = StringBuilder().append(model.main.humidity).append(" %").toString()  
    Glide.with(this).load(weatherImgUrl).into(binding.weatherImg)  
}
```
```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {  
    super.onViewCreated(view, savedInstanceState)
    ''''
    initWeatherView()
    observeData() 
}
```

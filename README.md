
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

#### HomeFragment.kt 날씨 띄우기

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
## 동물 감정 분석

앨범에서 고른 사진이나 카메라로 찍은 사진을 선택하여 해당 사진에 개, 고양이가 있다면 감정 분석 값을 받아볼 수 있다.
개, 고양이가 있는 사진을 선택해 감정을 분석해볼 수 있다.
* 사진을 앨범에서 고르거나 카메라로 찍어서 전송한다.
* 로딩 시간이 흐른 후 결과를 받아온다.
* 결과 값은 개, 고양이의 종과 화남, 행복, 슬픔의 감정을 퍼센트로 보내준다.

(감정 분석 사진 추가)

#### AnalysisFragment.kt

앨범, 카메라 접근 권한이 있는지 확인한다.
```kotlin
private fun getPermissions() {  
    if (ContextCompat.checkSelfPermission(  
            requireContext(),  
  Manifest.permission.CAMERA  
  )  
        != PackageManager.PERMISSION_GRANTED &&  
        ContextCompat.checkSelfPermission(  
            requireContext(),  
  Manifest.permission.WRITE_EXTERNAL_STORAGE  
  )  
        != PackageManager.PERMISSION_GRANTED &&  
        ContextCompat.checkSelfPermission(  
            requireContext(),  
  Manifest.permission.READ_EXTERNAL_STORAGE  
  )  
        != PackageManager.PERMISSION_GRANTED  
  ) {  
        ActivityCompat.requestPermissions(  
            requireActivity(),  
  PERMISSIONS,  
  PERMISSIONS_REQUEST  
  )  
    }  
}
```

앨범 또는 카메라를 선택한다. 
다음 ResultFragment 에서 무엇을 선택했는지 알기 위해 navArgs을 이용한다.
```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {  
    super.onViewCreated(view, savedInstanceState)  
    getPermissions()  
    binding.takePicture.setOnClickListener {  
    var action = AnalysisFragmentDirections.actionAlbumFragmentToResultFragment(1) //앨범 선택 1
    findNavController().navigate(action)  
    } 
    binding.selectPicture.setOnClickListener {  
    var action = AnalysisFragmentDirections.actionAlbumFragmentToResultFragment(2) //카메라 선택 2
    findNavController().navigate(action)  
    }  
}
```

#### ResultFragment.kt

```kotlin
```

사진을 서버로 전송하기 위해 Retrofit을 이용한다.
```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {  
    super.onViewCreated(view, savedInstanceState)  
    var gson = GsonBuilder().setLenient().create()  
    val retrofit = Retrofit.Builder()  
        .baseUrl("http://ec2-54-180-166-236.ap-northeast-2.compute.amazonaws.com:8080/")  
        .client(client)  
        .addConverterFactory(GsonConverterFactory.create(gson))  
        .build()  
    analysisApi = retrofit.create(AnalysisService::class.java)  
}
```

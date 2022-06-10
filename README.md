# SampleApplication

## よくある気をつけるべきこと
- 公式ドキュメントに記載があるかと言ってライブラリのバージョンが最新とは限らないので、導入の際は一度調べる
- 


## ViewBinding
### View Bindingの有効化
appフォルダ配下の「build.gradle」でView Bindingを有効化する。
```
android {
    // 省略
    
    // ViewBindingのセットアップ
    viewBinding {
        enabled true
    }
}
```

### FragmentでViewBindingを使用する
`onCreate`の前に下記を記載する。
```
class MainFragment : Fragment() {
    private var _binding: FragmentMainBinding? = null
    private val binding get() = _binding

}
```
```
override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        return inflater.inflate(R.layout.monster_detail_fragment, container, false)
        // 上記を削除し、下記に書き換える
        _binding = FragmentMainBinding.inflate(inflater, container, false)
        return binding?.root
    }
```
```
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        val alertDialogButton: Button = view.findViewById(R.id.alert_dialog_button)
        // 上記をViewBindingを利用し書き換えた形が下記になる
        binding?.alertDialogButton?.setOnClickListener {

        }
    }
```
```
override fun onDestroy() {
        super.onDestroy()

        // 下記を追加する
        _binding = null
    }
``` 


## Kotlin Coroutine
- コルーチンとは、Androidで使用できる並行実行のデザインパターンである
- Kotlinを開発したJetBrains社によって提供されている非同期処理ライブラリ
- Androidでの非同期プログラミングに推奨するソリューション
- 非同期実行するコードを簡略化


### 機能
- 軽量な非同期処理
- 非同期処理の実装はGoogleが提供するThreadやAsyncTaskなどのクラスで実装することもできる
- 非同期処理ごとにクラスを作成する必要がない
- 作成したクラスが残ってしまうことによってリークを起こすことがない
  - ThreadやAsyncTaskを使用する場合、作成したクラスからライフサイクルが終了したActivityやViewModel内のメソッドを呼び出されリークするリスクが少なからず存在する
- 

### 依存関係情報
- appフォルダ配下の「build.gradle」で`kotlinx-coroutines`のライブラリを追加する
- [GitHub:kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines)をREADMEを確認すると、最新のバージョンを確認できる
  - [Android での Kotlin コルーチン](https://developer.android.com/kotlin/coroutines?hl=ja)では最新でない場合があるので注意する

```
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.6.2")
}
```

### CoroutineScope
#### ライフサイクルに対応するCoroutineScope
##### ViewModelScope
- ViewModelScope は、アプリで ViewModel ごとに定義される
- このスコープ内で起動されたすべてのコルーチンは、ViewModel が消去されると自動的にキャンセルされる
```
class MyViewModel: ViewModel() {
    init {
        viewModelScope.launch {
            // Coroutine that will be canceled when the ViewModel is cleared.
        }
    }
}
```

##### LifecycleScope
- LifecycleScope は、Lifecycle オブジェクトごとに定義される
  - この場合ActivityやFragment内で定義される
- このスコープ内で起動されたすべてのコルーチンは、Lifecycle が破棄されたときにキャンセルされる
- LifecycleのCoroutineScopeは下記のようにプロパティを介してアクセスする
  - Activityでは`lifecycle.coroutineScope` プロパティ
  - Fagmentでは`lifecycleOwner.lifecycleScope` プロパティ

```
class MyActivity: AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        lifecycle.coroutineScope.launch {

        }
}
```
```
class MyFragment: Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewLifecycleOwner.lifecycleScope.launch {
            val params = TextViewCompat.getTextMetricsParams(textView)
            val precomputedText = withContext(Dispatchers.Default) {
                PrecomputedTextCompat.create(longTextContent, params)
            }
            TextViewCompat.setPrecomputedText(textView, precomputedText)
        }
    }
}
```

##### 再実行可能なライフサイクル対応コルーチン
- 長時間実行オペレーションは、Lifecycle が DESTROYED の場合、lifecycleScope により適切な方法で自動的にキャンセルされる
- Lifecycle が特定の状態のときにコードブロックの実行の開始が、また別の状態のときにキャンセルが必要な場合もある
  - たとえば、Lifecycle が STARTED のときに Flow を収集し、STOPPED になるとコレクションをキャンセルする必要がある場
- UI が画面に表示されるときにのみ Flow 出力を処理することで、リソースを節約し、場合によってはアプリのクラッシュを回避する
- Lifecycle と LifecycleOwner にはこれを実現する suspend repeatOnLifecycle API が用意されています。次の例には、関連する Lifecycle が少なくとも STARTED 状態にあるたびに実行し、Lifecycle が STOPPED になるとキャンセルするコードブロックが含まれる

```
class MyFragment : Fragment() {

    val viewModel: MyViewModel by viewModel()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // Create a new coroutine in the lifecycleScope
        viewLifecycleOwner.lifecycleScope.launch {
            // repeatOnLifecycle launches the block in a new coroutine every time the
            // lifecycle is in the STARTED state (or above) and cancels it when it's STOPPED.
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                // Trigger the flow and start listening for values.
                // This happens when lifecycle is STARTED and stops
                // collecting when the lifecycle is STOPPED
                viewModel.someDataFlow.collect {
                    // Process item
                }
            }
        }
    }
}
```

**ライフサイクル対応フロー収集**
- ライフサイクル対応収集を単一のフローで行うだけでよい場合は、Flow.flowWithLifecycle() メソッドを使用してコードを簡略化できる

```
viewLifecycleOwner.lifecycleScope.launch {
    exampleProvider.exampleFlow()
        .flowWithLifecycle(viewLifecycleOwner.lifecycle, Lifecycle.State.STARTED)
        .collect {
            // Process the value.
        }
}
```

- 複数のフローで並行してライフサイクル対応収集を行う必要がある場合は、各フローを別々のコルーチンで収集する必要があります。その場合は、repeatOnLifecycle() を直接使用すると効率的

```
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        // Because collect is a suspend function, if you want to
        // collect multiple flows in parallel, you need to do so in
        // different coroutines.
        launch {
            flow1.collect { /* Process the value. */ }
        }

        launch {
            flow2.collect { /* Process the value. */ }
        }
    }
}
```

##### ライフサイクル対応コルーチンを停止する
Lifecycle が特定の状態でない限り、コードブロックの実行の停止が必要な場合もある
FragmentTransaction を実行するには、Lifecycle が少なくとも STARTED になるまで待つ必要がある
Lifecycle では、lifecycle.whenCreated、lifecycle.whenStarted、lifecycle.whenResumed などの追加メソッドを提供します。Lifecycle が必要最小限の状態にない場合、これらのブロック内で実行されているすべてのコルーチンが停止する

```
class MyFragment: Fragment {
    init { // Notice that we can safely launch in the constructor of the Fragment.
        lifecycleScope.launch {
            whenStarted {
                // The block inside will run only when Lifecycle is at least STARTED.
                // It will start executing when fragment is started and
                // can call other suspend methods.
                loadingView.visibility = View.VISIBLE
                val canAccess = withContext(Dispatchers.IO) {
                    checkUserAccess()
                }

                // When checkUserAccess returns, the next line is automatically
                // suspended if the Lifecycle is not *at least* STARTED.
                // We could safely run fragment transactions because we know the
                // code won't run unless the lifecycle is at least STARTED.
                loadingView.visibility = View.GONE
                if (canAccess == false) {
                    findNavController().popBackStack()
                } else {
                    showContent()
                }
            }

            // This line runs only after the whenStarted block above has completed.

        }
    }
}
```

### 参考サイト
- [Android での Kotlin コルーチン](https://developer.android.com/kotlin/coroutines?hl=ja)
- [Coroutines guide](https://kotlinlang.org/docs/coroutines-guide.html)
- [GitHub:kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines)
- [ライフサイクル対応コンポーネントで Kotlin コルーチンを使用する](https://developer.android.com/topic/libraries/architecture/coroutines#viewmodelscope)
- 


## FirebaseMessage

### 必要ライブラリ
- [WorkManager](https://developer.android.com/jetpack/androidx/releases/work?hl=ja)
- 
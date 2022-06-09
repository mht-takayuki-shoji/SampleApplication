# SampleApplication

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


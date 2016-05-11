# 谈谈 MVVM 设计模式在 Unity 3D 中的设计与实现 #
谈起 MVVM 设计模式，可能第一映像你会想到 WPF/Sliverlight，他们提供了的数据绑定（Data Binding），命令（Command）等功能，这让 MVVM 模式得到很好的实现。

MVVM 设计模式顾名思义，通过分离关注点，各司其职。通过 Data Binding 可达到数据的双向绑定，而命令 Command 更是将传统的 Code Behind 事件独立到 ViewModel 中。

![](http://images.cnblogs.com/cnblogs_com/OceanEyes/354373/o_bb.png)
## MVVM 设计模式在 WPF 中的实现 ##

在WPF中，你会像如下这样去定义一个专门管理视图 View 的 ViewModel：

    public class SongViewModel : INotifyPropertyChanged
    {
        #region Construction
        /// Constructs the default instance of a SongViewModel
        public SongViewModel()
        {
            _song = new Song { ArtistName = "Unknown", SongTitle = "Unknown" };
        }
        #endregion
 
        #region Members
        Song _song;
        #endregion
 
        #region Properties
        public Song Song
        {
            get
            {
                return _song;
            }
            set
            {
                _song = value;
            }
        }
 
        public string ArtistName
        {
            get { return Song.ArtistName; }
            set
            {
                if (Song.ArtistName != value)
                {
                    Song.ArtistName = value;
                    RaisePropertyChanged("ArtistName");
                }
            }
        }
        #endregion
 
        #region INotifyPropertyChanged Members
 
        public event PropertyChangedEventHandler PropertyChanged;
 
        #endregion
 
        #region Methods
 
        private void RaisePropertyChanged(string propertyName)
        {
            // take a copy to prevent thread issues
            PropertyChangedEventHandler handler = PropertyChanged;
            if (handler != null)
            {
                handler(this, new PropertyChangedEventArgs(propertyName));
            }
        }
        #endregion
    }

同时在 View 中你需要使用 Binding 将 ViewModel 的属性绑定和控件的内容相绑定：

     <TextBox Content="{Binding ArtistName}" />

值得注意的是，要实现 View 和 ViewModel 双向绑定，我们的 ViewModel 必须实现 **INotifyPropertyChanged** 接口，由于 WPF Framework 让控件监听了 PropertyChanged 事件，当属性值发生时，触发 PropertyChanged 事件，所以控件就能自动获取到最新的值。反之，当控件的值发生改变时，例如 TextBox 触发 OnTextChanged 事件，自动将最新的值同步到 ViewModel 相应的属性中。

## MVP & MVVM ##

Unity 3D 与 WPF/Sliverlight 不同，它没有提供类似的 Data Binding，也没有像 XAML 一样的视图语法，那么怎样才能在 Unity 3D 中去实现 MVVM 呢？

在 ASP.NET WebForm 时代，那时还没有 ASP.Net MVC 。我们为了让 UI 表现层分离，常常会使用 MVP 设计模式，以下是我在几年前画的一张老图：
![](http://images.cnblogs.com/cnblogs_com/OceanEyes/354373/o_aaa.jpg)

> MVP 设计模式核心就是，通过定义一个 View，将 UI 抽象出来，它不必关心数据的具体来源，也不必关心点击按钮之后业务逻辑的实现，它只关注 UI 交互。这就是典型的分离关注点。

其实这就是我今天想讲的主题，既然 Unity 3D 没有提供数据绑定，那么我们也可以参考之前 MVP 的设计理念：

> 将 UI 抽象成独立的一个个 View，将面向 Component 开发转换为面向 View 开发，每一个 View 都有独立的 ViewModel 进行管理，如下所示：

![](http://images.cnblogs.com/cnblogs_com/OceanEyes/354373/o_QQ%e5%9b%be%e7%89%8720160511122638.jpg)

由于 Unity 3D 没有 XAML,也没有 Data Binding 技术，故只能在抽象出来的 View 中去实现类似于 WPF 的 Data Binding，Converter，Command 等。

> 值得注意的是，MVP 设计模式中数据的绑定是通过将具体的 View 实例传递到 Presenter 中完成的，而 MVVM 是以数据改变引发的事件中完成数据更新的。

## MVVM 设计模式在 Unity 3D 中的设计与实现 ##

再回顾一下 WPF 中 ViewModel 的写法。 ViewModel 提供了 View 需要的数据，并且 ViewModel  实现 **INotifyPropertyChanged** 接口 ，当数据更改时，触发了 **PropertyChanged** 事件，由于控件也监听了此事件，在事件的响应函数里实现数据的更新。

了解了之后，我们要考虑怎样在 Unity 3D 中去实现它。假设我们需要完成如下的一个功能，并且是使用 MVVM 设计思想实现：

![](http://images.cnblogs.com/cnblogs_com/OceanEyes/354373/o_3VyWDn1Gj0%20(1).gif)

首先，我们要定义一个 View，这个 View 是对 UI 元素的一个抽象，到底要抽象哪些 UI 元素呢？就这个例子而言，InputField，Label，Slider，Toggle，Button 是需要被抽象出来的。
    
    public class SetupView
    {
    	public InputField nameInputField;
   	    public Text nameMessageText;

        public InputField jobInputField;
	    public Text jobMessageText;
	    
	    public InputField atkInputField;
	    public Text atkMessageText;
	    
	    public Slider successRateSlider;
	    public Text successRateMessageText;
	    
	    public Toggle joinToggle;
	    public Button joinInButton;
	    public Button waitButton;
    }

可以看到，这是一个很简单的 View。接着我们需要定义一个专门用来管理 View 的 ViewModel，它以属性的形式提供数据，以方法的形式提供行为。

值得注意的是，ViewModel 中的属性不是特殊的属性，它必须具备当数据更改时通知订阅者这个功能，怎么通知订阅者？当然是事件，故我把此属性称为 *BindableProperty* 属性。

     public class BindableProperty<T>
    {
        public delegate void ValueChangedHandler(T oldValue, T newValue);

        public ValueChangedHandler OnValueChanged;

        private T _value;
        public T Value
        {
            get
            {
                return _value;
            }
            set
            {
                if (!object.Equals(_value, value))
                {
                    T old = _value;
                    _value = value;
                    ValueChanged(old, _value);
                }
            }
        }

        private void ValueChanged(T oldValue, T newValue)
        {
            if (OnValueChanged != null)
            {
                OnValueChanged(oldValue, newValue);
            }
        }

        public override string ToString()
        {
            return (Value != null ? Value.ToString() : "null");
        }
    }

接着，我们再定义一个 ViewModel，它为 View 提供了数据和行为:

     public class SetupViewModel : ViewModel
    {
        public BindableProperty<string> Name = new BindableProperty<string>();
        public BindableProperty<string> Job = new BindableProperty<string>();
        public BindableProperty<int> ATK = new BindableProperty<int>();
        public BindableProperty<float> SuccessRate = new BindableProperty<float>();
        public BindableProperty<State> State = new BindableProperty<State>();
    }

有了 View 与 ViewModel 之后，我们需要考虑：

- 怎样为 View 指定一个 ViewModel
- 当 ViewModel 属性值改变时，怎样订阅触发的 OnValueChanged 事件，从而达到 View 的数据更新

基于以上两点，我们可以定义一个通用的 View，将它命名为 **UnityGuiView** ：

    public interface IView
    {
        ViewModel BindingContext { get; set; }
    }

    public class UnityGuiView:MonoBehaviour,IView
    {
        public readonly BindableProperty<ViewModel> ViewModelProperty = new BindableProperty<ViewModel>();
        public ViewModel BindingContext
        {
            get { return ViewModelProperty.Value; }
            set { ViewModelProperty.Value = value; }
        }

        protected virtual void OnBindingContextChanged(ViewModel oldViewModel, ViewModel newViewModel)
        {
        }

        public UnityGuiView()
        {
            this.ViewModelProperty.OnValueChanged += OnBindingContextChanged;
        }

    }

- 上述代码中，提供一个 BindingContext 上下文属性，类似于 WPF 中的 DataContext。 BindingContext 属性我们不能将它视为一个简单的属性 ，它是上述定义过的 BindableProperty 类型属性。那么当为一个 View 的 BindingContext 指定 ViewModel 实例时，初始化时，势必会触发 OnValueChanged 事件。

- 在响应函数 OnBindingContextChanged 中 ，我们可以在此对 ViewModel 中事件进行监听，从而达到数据的更新。当然这是一个虚方法，你需要在子类 View 中 Override。

所以修改定义过的 SetupView，继承自 UnityGuiView:

    public class SetupView:UnityGuiView
    {
       ...省略部分代码

       public SetupViewModel ViewModel { get { return (SetupViewModel)BindingContext; } }

       protected override void OnBindingContextChanged(ViewModel oldViewModel, ViewModel newViewModel)
        {

            base.OnBindingContextChanged(oldViewModel, newViewModel);

            SetupViewModel oldVm = oldViewModel as SetupViewModel;
            if (oldVm != null)
            {
                ViewModel.Name.OnValueChanged -= NameValueChanged;
                ...
            }
            if (ViewModel!=null)
            {
                ViewModel.Name.OnValueChanged += NameValueChanged;
                ...
            }
            UpdateControls();
        }

        private void NameValueChanged(string oldvalue, string newvalue)
        {
            nameMessageText.text = newvalue.ToString();
        }
    }
由于子类 Override 了 OnBindingContextChanged 方法，故它会对 ViewModel 的属性值改变事件进行监听，当触发时，将最新的数据同步到 UI 中。

同理，考虑到双向绑定，你也可以在 View 中定义一个 OnTextBoxValueChanged 响应函数，当文本框中的数据改变时，在响应函数中就数据同步到 ViewModel 中。在这我就不累述了。

最后，在 Unity 3D 中将 SetupView 附加到 相应的 GameObject上：

![](http://images.cnblogs.com/cnblogs_com/OceanEyes/354373/o_aaa.png)

最后在摄像机上加一段脚本，很简单，传入 SetupView 对象并为其绑定 ViewModel：

    public SetupView setupView;
    void Start()
    {
        //绑定上下文
        setupView.BindingContext=new SetupViewModel();
    }

## 小结 ##
这是一个非常简单的 MVVM 框架，也证明了在 Unity 3D 中实现 MVVM 设计模式的可能性。相应的代码可在 Github 上详细查看，地址为：[https://github.com/MEyes/uMVVM](https://github.com/MEyes/uMVVM)

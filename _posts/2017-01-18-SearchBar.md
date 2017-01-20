---
layout: post
title: "简单的Android带历史记录的搜索功能实现"
date: 2017-01-18
description: "简单的Android带历史记录的搜索功能实现"
tag: Android
---   

这两天闲来无事，就把前短时间项目中的搜索功能抽取出来，重新写一下，搜索功能虽然简单，但是涉及到的知识点也挺多的，就当做一个总结吧。
代码地址在最后面，话不多说，上效果图 ^_^
  ![searchdemo.gif](http://upload-images.jianshu.io/upload_images/1859111-be6ba876e6df7526.gif?imageMogr2/auto-orient/strip)

### 历史记录的存储

首先来说下关于历史记录的存储，历史记录的存储方式其实可以有很多方法，可以用sp，数据库等等，那么就直接开撸吧。说开撸你还真以为就直接开撸了，还是先想想吧，我们做历史记录存储的时候需要提供什么给调用者，其实很简单无非就是可以增删改查吗。为了遵守里氏替换原则，就先写了个抽象类BaseHistoryStorage，里面有几个抽象方法，至于是什么自己看吧。这样不管你是用SP还是数据库甚至其他更牛的技术，只需要继承Base类，实现这些抽象方法就行了，至于你是怎么实现的，我才不管呢。

```bash

/**
 * 历史信息存储基类
 * Created by Zellerpooh on 17/1/18.
 */

public abstract class BaseHistoryStorage {
    /**
     * 保存历史记录时调用
     *
     * @param value
     */
    public abstract void save(String value);

    /**
     * 移除单条历史记录
     *
     * @param key
     */
    public abstract void remove(String key);

    /**
     * 清空历史记录
     */
    public abstract void clear();

    /**
     * 生成key
     *
     * @return
     */
    public abstract String generateKey();

    /**
     * 返回排序好的历史记录
     *
     * @return
     */
    public abstract ArrayList<SearchHistoryModel> sortHistory();
}

```

上面的代码很好理解，SearchHistoryModel是我写的一个JavaBean,里面就放了两个String,一个是历史搜索的内容，一个是历史记录的Key，其实你直接返回一个String泛型的ArrayList的就行，但是我这里为了用SP实现的时候跟快速偷了个懒，好了自己去实现一个历史记录存储功能把。


####   通过SharedPreference实现数据存储

听到让你自己去实现是不是心凉了一半，当然是逗你的了，既然都来了怎么能不给你点福利呢，下面我就实现一个简单的通过SharedPreference实现的数据存储吧，来抛砖迎玉吧。

```bash

 private static SpHistoryStorage instance;

 private SpHistoryStorage(Context context, int historyMax) {
        this.context = context.getApplicationContext();
        this.HISTORY_MAX = historyMax;
    }

 public static synchronized SpHistoryStorage getInstance(Context context, int historyMax) {
        if (instance == null) {
            synchronized (SpHistoryStorage.class) {
                if (instance == null) {
                    instance = new SpHistoryStorage(context, historyMax);
                }
            }
        }
        return instance;
    }

```

   作为一个励志成为高逼格的高级程序员的菜鸟，当然不会放过任何装逼的机会，对于这种比较耗资源的数据存储，将他设计为单例模式当然最合适不过了，上面就是一个简单的DCL的单例模式实现,在安卓中将Context传入到单例中是一个大忌，你试想一下你的activity永远得不到释放是一件多么恐怖的事情，所以我就换成了applicationContext,反正他是跟随程序一直在的。
    然后来看看里面的方法实现吧

```bash

 private static SimpleDateFormat mFormat = new SimpleDateFormat("yyyyMMddHHmmss");
 @Override
    public String generateKey() {
        return mFormat.format(new Date());
    }

```

上面这个方法就是为了存储的时候根据当前时间生成的一个Key,用来判断先后顺序。

```bash

    @Override
    public ArrayList<SearchHistoryModel> sortHistory() {
        Map<String, ?> allHistory = getAll();
        ArrayList<SearchHistoryModel> mResults = new ArrayList<>();
        Map<String, String> hisAll = (Map<String, String>) getAll();
        //将key排序升序
        Object[] keys = hisAll.keySet().toArray();
        Arrays.sort(keys);
        int keyLeng = keys.length;
        //这里计算 如果历史记录条数是大于 可以显示的最大条数，则用最大条数做循环条件，防止历史记录条数-最大条数为负值，数组越界
        int hisLeng = keyLeng > HISTORY_MAX ? HISTORY_MAX : keyLeng;
        for (int i = 1; i <= hisLeng; i++) {
            mResults.add(new SearchHistoryModel((String) keys[keyLeng - i], hisAll.get(keys[keyLeng - i])));
        }
        return mResults;
    }

 public Map<String, ?> getAll() {
        SharedPreferences sp = context.getSharedPreferences(SEARCH_HISTORY,
                Context.MODE_PRIVATE);
        return sp.getAll();
    }

```

这个方法就是从SharedPreferences取出所有的值并且按照时间先后进行排序。

```bash

  @Override
    public void save(String value) {
        Map<String, String> historys = (Map<String, String>) getAll();
        for (Map.Entry<String, String> entry : historys.entrySet()) {
            if (value.equals(entry.getValue())) {
                remove(entry.getKey());
            }
        }
        put(generateKey(), value);
    }

```

保存的方法，需要先判断是否已经存在，存在的话就先删除然后根据最新的时间保存。剩下两个移除和清空的方法就自己看吧。

```bash

 @Override
    public void remove(String key) {
        SharedPreferences sp = context.getSharedPreferences(SEARCH_HISTORY,
                Context.MODE_PRIVATE);
        SharedPreferences.Editor editor = sp.edit();
        editor.remove(key);
        editor.commit();
    }

  @Override
 public void clear() {
        SharedPreferences sp = context.getSharedPreferences(SEARCH_HISTORY,
                Context.MODE_PRIVATE);
        SharedPreferences.Editor editor = sp.edit();
        editor.clear();
        editor.commit();
    }

```

 好了，通过sharedPreferences存储历史记录的功能就这样完成了，但是心里一想好久没有写数据库相关的代码了，就简单的撸了一个通过数据库存储的方法，篇幅有限代码就不贴了，想看的[点这里](https://github.com/Zellerpooh/FastLibrary/blob/master/app/src/main/java/com/zeller/fastlibrary/searchhistory/storage/DbHistoryStorage.java)。其实这里的代码借鉴了[remusic](https://github.com/aa112901/remusic)的搜索记录存储的实现，有什么问题找他去→_→。

###  界面的实现

界面的实现就比较朴素了，毕竟我们都是比较注重内在的人，代码就不贴了，上面一个EditText,下面一个ListView再带上一个清空的按钮。界面写好了之后呢，先给ListView撸个adapter吧，也很简单继承BaseAdapter,实现下方法，然后暴露个接口出来，用于单条历史记录被点击和删除的时候用。

```bash

public interface OnSearchHistoryListener {
    void onDelete(String key);

    void onSelect(String content);
}
 holder.layoutClose.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (onSearchHistoryListener != null) {
                    onSearchHistoryListener.onDelete(mHistories.get(position).getTime());
                }
            }
        });
        holder.layout.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (onSearchHistoryListener != null) {
                    onSearchHistoryListener.onSelect(mHistories.get(position).getContent());
                }
            }
        });

```

###  数据绑定
界面撸完了，接着就需要将数据和界面联系起来了，这里就不得不再推荐一下MVP模式了，Model和View都由中间层Presenter来控制，使得逻辑看起来变得很清晰，所以这里就用MVP模式写了，其实我是怕自己把各位绕糊涂。

```bash

public interface SearchPresenter {

    void remove(String key);

    void clear();

    void sortHistory();

    void search(String value);
}

public interface SearchView {
    void showHistories(ArrayList<SearchHistoryModel> results);

    void searchSuccess(String value);
}

public interface SearchModel {

    void save(String value);

    void search(String value,OnSearchListener onSearchListener);

    void remove(String key);

    void clear();

    void sortHistory(OnSearchListener onSearchListener);
}

```

presenter要做的事情就是，当View被操作后，通知它去做数据操作，即增删改查，而Model只需要在presenter告诉它做什么的时候去做就行，成功之后再回调presenter去通知View,这里View就只需要两个操作，搜索成功后界面的切换以及历史记录的显示。那接下来就来实现一下model,presenter和View。

```bash

public class SearchPresenterImpl implements SearchPresenter, OnSearchListener             {
    private static final int historyMax = 10;
    private SearchView searchView;
    private SearchModel searchModel;

    public SearchPresenterImpl(SearchView searchView, Context context) {
        this.searchView = searchView;
        this.searchModel = new SearchModelImpl(context, historyMax);
    }

    //移除历史记录
    @Override
    public void remove(String key) {
        searchModel.remove(key);
        searchModel.sortHistory(this);
    }

    @Override
    public void clear() {
        searchModel.clear();
        searchModel.sortHistory(this);
    }

    //获取所有的历史记录
    @Override
    public void sortHistory() {
        searchModel.sortHistory(this);
    }

    @Override
    public void search(String value) {
        searchModel.save(value);
        searchModel.search(value, this);
    }

    @Override
    public void onSortSuccess(ArrayList<SearchHistoryModel> results) {
        searchView.showHistories(results);
    }

    @Override
    public void searchSuccess(String value) {
        searchView.searchSuccess(value);
    }
}

```

在初始化presenter的同时引用了View和Model,然后实现OnSearchListener当model完成操作是回调view中的方法。代码自己看吧，应该没有任何疑问。

```bash

public class SearchModelImpl implements SearchModel {

    private BaseHistoryStorage historyStorage;

    public SearchModelImpl(Context context, int historyMax) {
//        historyStorage = SpHistoryStorage.getInstance(context, historyMax);
        historyStorage = DbHistoryStorage.getInstance(context, historyMax);
    }

    @Override
    public void save(String value) {
        historyStorage.save(value);
    }

    @Override
    public void search(String value, OnSearchListener onSearchListener) {
        onSearchListener.searchSuccess(value);
    }

    @Override
    public void remove(String key) {
        historyStorage.remove(key);
    }

    @Override
    public void clear() {
        historyStorage.clear();
    }

    @Override
    public void sortHistory(OnSearchListener onSearchListener) {
        onSearchListener.onSortSuccess(historyStorage.sortHistory());
    }
}

```

model中的内容就简单了，创建一个前面实现的BaseStorage对象，对数据进行操作，这里没有对搜索做什么处理直接通过回调返回了传进来的字符串，在实际开发中应该是去请求接口，返回参数，所以在View中也没有做具体的处理，实际开发中可以打开一个新的页面后者，切换列表显示搜索到的内容。

```bash

 @Override
    public void showHistories(ArrayList<SearchHistoryModel> results) {
        llSearchEmpty.setVisibility(0 != results.size() ? View.VISIBLE : View.GONE);
        searchHistoryAdapter.refreshData(results);
    }

    @Override
    public void searchSuccess(String value) {
        Toast.makeText(this, value, Toast.LENGTH_SHORT).show();
    }
```

上面就是View中实现的方法，获得历史记录是，告诉adapter去刷新列表就行了。接下来就只剩下View中一些简单的点击事件的处理了，搜索的时候调用```
 mSearchPresenter.search(value);```,清空的时候调用``` mSearchPresenter.clear();```是不是感觉so easy，妈妈再也不用担心我的学习了，当然别忘了presenter需要在activity的onCreate方法中进行实例化。
 
最后呢，再给大家介绍几个技巧：

1.  不要忘记把搜索框EditText设置成Search模式     

```bash

android:imeOptions="actionSearch"

```
设置完以后不要忘记对键盘上的搜索按钮的监听

```bash

etSearch.setOnEditorActionListener(new TextView.OnEditorActionListener() {
            @Override
            public boolean onEditorAction(TextView v, int actionId, KeyEvent event) {
                if (actionId == EditorInfo.IME_ACTION_SEARCH) {
                    search(etSearch.getText().toString());
                    return true;
                }
                return false;
            }
        });

```

2.  我们看到很多软件都会做模糊搜索的操作，你一输入列表就会弹出很多相关的词汇供你点击，顿时感觉好贴心哦，其实这个功能要实现也不能，通过给editText设置监听以及Handler延时发送消息就能够实现了。

```bash

    private TextWatcher textWatcher = new TextWatcher() {
        @Override
        public void beforeTextChanged(CharSequence s, int start, int count, int after) {

        }

        @Override
        public void onTextChanged(CharSequence s, int start, int before, int count) {

        }

        @Override
        public void afterTextChanged(Editable s) {
            //在500毫秒内改变时不发送
            if (mHandler.hasMessages(MSG_SEARCH)) {
                mHandler.removeMessages(MSG_SEARCH);
            }
            if (TextUtils.isEmpty(s)) {
                llSearchHistory.setVisibility(View.VISIBLE);
                mSearchPresenter.sortHistory();
            } else {
                llSearchHistory.setVisibility(View.GONE);
                //否则延迟500ms开始模糊搜索
                Message message = mHandler.obtainMessage();
                message.obj = s;
                message.what = MSG_SEARCH;
                mHandler.sendMessageDelayed(message, 500); //自动搜索功能 删除
            }
        } }；
        //模糊搜索
        private static final int MSG_SEARCH = 1;
        private Handler mHandler = new Handler() {
          @Override
          public void handleMessage(Message msg) {
            search(etSearch.getText().toString().trim());
          }  
        };

```

啰嗦了这么半天，最后还是附上代码吧，[searchBar](https://github.com/Zellerpooh/FastLibrary/blob/master/app/src/main/java/com/zeller/fastlibrary/searchhistory)
有什么Bug和可以优化的地方还是希望大家能够留言，你们都将是菜鸡成长路上的恩师。

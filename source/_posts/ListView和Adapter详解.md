title: ListView和Adapter详解
date: 2015-09-22 15:13:06
tags: ["Android"]
---
## 什么是ListView
> ListView is a view group that displays a list of scrollable items. The list items are automatically inserted to the list using an Adapter that pulls content from a source such as an array or database query and converts each item result into a view that’s placed into the list.

其实很容易理解，ListView就是一个能数据集合以动态滚动的方式展示到用户界面上的View。
<!--more-->

## ListView和Adapter

ListView和数据是分开的，不直接接触。而是通过Adapter(适配器)加载到屏幕。也就是说Adapter相当于是数据和View之间的桥梁。Adapter负责为每一个数据项制作View。

![ListView原理](http://ww4.sinaimg.cn/mw690/87dacc16gw1ewbbhro6s3j20dr0agwfb.jpg)

## ListView工作原理

在运行时ListView会针对数据项向Adapter要View，从而来加载到界面上。那么，难道每一个item，Adapter都会重新创建一个视图吗？答案很明显是不可能的。

为了能够更快更省空间的显示，Android准备了一个叫做Recycler(循环)的组件。假如你的屏幕只能显示七个item。那么ListView只会创建八个item的视图。当第一个item出去，离开屏幕的时候，这个item的view就会被拿来重用，显示下一个item的内容。

![Recycler](http://android.amberfog.com/wp-content/uploads/2010/02/listview_recycler.jpg)

## 继承BaseAdapter来使用ListView

BaseAdapter是个抽象类，继承它能很方便的来使用ListView。要覆写四个方法。前三个都很简单。重点是getView方法，这直接决定你的屏幕加载是否顺滑。

1.  如果convertView为空，加载布局创建View
2.  若不空，则直接通过position，获取list中的item，直接替换view中的内容。

很清楚了吧？convertView就是一个View缓冲器。这里的getview的demo不是很好。因为findViewById()这个函数执行代价很大。所以不建议用这种方式，每次都加载控件。
```java
public class MyDataAdapter extends BaseAdapter {

    private static final String TAG = "MyDataAdapter";

    private static int counter = 0;

    //数据源
    List<MyDataClass> mData = null;
    Context mContext = null;


    public MyDataAdapter(Context context, List<MyDataClass> data) {
        mContext = context;
        mData = data;
        counter = 0;
    }

    //数据源中的数据对象个数
    @Override
    public int getCount() {
        return mData.size();
    }

    //指定位置处的数据源对象
    @Override
    public Object getItem(int position) {
        return mData.get(position);
    }

    //数据源对象的Id,如果有的话
    //如果数据源对象自己没有定义Id，则可以简单地返回其在数据源中的位置
    @Override
    public long getItemId(int position) {
        return position;
    }


    //每当Android ListView需要显示一行时，它会调用此方法
    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        Log.d(TAG, "显示：" + position + "行，调用getView()" + "参数convertView==null?" + (convertView == null));

        View rootView = null;
        //如果没有可以重用的控件
        if (convertView == null) {
            LayoutInflater inflater = LayoutInflater.from(mContext);
            rootView = inflater.inflate(R.layout.my_list_item, parent, false); //加载布局，创建View
            rootView.setTag(position);
            Log.d(TAG, "实例化rootView，保存到Tag中的值为:" + position);
            counter++;
            Log.d(TAG, "控件实例化个数：" + counter);
        } else {
            //控件己经被创建过，直接重用
            rootView = convertView;
        }

        //依据位置提取相应的数据源对象
        MyDataClass item = mData.get(position);

        //获取用于显示内容的控件的引用
        TextView titleTextView = (TextView) rootView.findViewById(R.id.tvTitle);
        TextView detailTextView = (TextView) rootView.findViewById(R.id.tvDetail);

        //设置显示内容
        titleTextView.setText(item.getTitle());
        detailTextView.setText(item.getDetail() + " 本行Tag中保存的值为:" + rootView.getTag());

        return rootView;
    }
}
```

## ViewHolder设计模式

创建一个内部类ViewHolder保存控件。只加载View的时候使用findViewById()方法。使用View的setTag()方法保存ViewHolder。当convertView不为空的时候，直接取出来即可
。
```java
    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        ViewHolder holder=null;
        View rootView=convertView;
        if(convertView==null){
            LayoutInflater layoutInflater = LayoutInflater.from(context);
            rootView = layoutInflater.inflate(R.layout.phoneinfo_item, parent, false);
            holder=new ViewHolder();
            holder.nameText= (TextView) rootView.findViewById(R.id.nameText);
            holder.numberText= (TextView) rootView.findViewById(R.id.numberText);
            rootView.setTag(holder);
        }
        else {
            holder= (ViewHolder) rootView.getTag();
        }
        PhoneInfo phoneInfo = phoneInfoList.get(position);
        holder.nameText.setText(phoneInfo.getPhoneName());
        holder.numberText.setText(phoneInfo.getPhoneNumber());

        return rootView;
    }

    private static class ViewHolder {
        public TextView nameText =null;
        public TextView numberText=null;
    }
```

## 含有多种视图的布局ListView

这应该是一种比较常见的情况。

如果你要加载多种布局那么**首先你一定重写这两个方法**

**注意！type一定要从0开始！！**
```java
    @Override
    public int getItemViewType(int position) {
        BaseModel baseModel=models.get(position);
        int type=baseModel.getType();
        return type;
    }


    @Override
    public int getViewTypeCount() {
        return 2;
    }
```

剩下的重点getView()其实也很简单啦，也就是弄多个ViewHolder保存即可
```java
    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        BaseModel baseModel=models.get(position);
        int type=getItemViewType(position);
        HeaderViewHolder headerViewHolder=null;
        DetailViewHolder detailViewHolder=null;
        if(convertView==null){
            LayoutInflater layoutInflater=LayoutInflater.from(context);
            if(type==0){
                headerViewHolder=new HeaderViewHolder();
                convertView=layoutInflater.inflate(R.layout.header_item,null);
                headerViewHolder.headerText= (TextView) convertView.findViewById(R.id.headerText);
                convertView.setTag(headerViewHolder);
            }
            else if(type==1){
                detailViewHolder=new DetailViewHolder();
                convertView=layoutInflater.inflate(R.layout.detail_item,null);
                detailViewHolder.contentText= (TextView) convertView.findViewById(R.id.contentText);
                detailViewHolder.detailText=(TextView) convertView.findViewById(R.id.detailText);
                detailViewHolder.clickBtn= (Button) convertView.findViewById(R.id.clickButton);
                convertView.setTag(detailViewHolder);
            }
        }
        else  {
            if(type==0)
                headerViewHolder= (HeaderViewHolder) convertView.getTag();
            else if(type==1)
                detailViewHolder= (DetailViewHolder) convertView.getTag();
        }

        if(type==0){
            HeaderModel headerModel= (HeaderModel) baseModel;
            headerViewHolder.headerText.setText(headerModel.getHeader());
        }
        else if(type==1){
            final DetailModel detailModel= (DetailModel) baseModel;
            detailViewHolder.contentText.setText(detailModel.getContext());
            detailViewHolder.detailText.setText(detailModel.getDetail());
            detailViewHolder.clickBtn.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    Toast.makeText(context,detailModel.getContext()+"被点击",Toast.LENGTH_SHORT).show();
                }
            });
        }
        return convertView;
    }

    private static class HeaderViewHolder {
        TextView headerText=null;
    }

    private static class DetailViewHolder {
        TextView contentText=null;
        TextView detailText=null;
        Button clickBtn=null;
    }
```

实在弄不清楚可以看我的github项目 [ListViewEventDemo](https://github.com/threezj/AndroidSimpleProject/tree/master/ListViewEventDemo)

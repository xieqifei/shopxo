# 1：内容梗概

ShopXO是款非常优秀的开源商城软件。为了让商城更加适合没有网站管理经验的人使用，我优化了手机和电脑端的商城后台。修复了小程序的一些BUG，并做了一些其他优化。使用飞鹅打印机，接收新订单推送通知和订单打印。利用微信认证的公众号，在有新订单时，通过公众号给微信推送新订单消息。

特别说明：此篇文章建议有一定前后端基础的朋友阅读，否则，你在按照本文修改你的网站时，可能由于我在写作时，将某个环节遗漏，而导致你的网站无法做到像我修改后一样的结果。此时，会得不偿失。同时，若你对本文内容有任何疑问，欢迎你通过邮箱与我联系。邮箱几乎每天一看。

在进行下面修改时，建议你先将你的网站压缩打包，保存一下。避免操作失误无法修复。如果你的修改在达到理想效果后，建议你都进行一下保存操作

# 2：商城后台优化

## 2.1 优化说明

> 优化说明：
>
> 1. 手机端显示商品缩率图
> 2. 直接点击价格，弹出价格修改框，快速改价。
> 3. 隐藏“查看更多”按钮。（给缩略图让空间）

对比：

![](https://ae01.alicdn.com/kf/H00d9341b2269430e9a8655a814b116a8g.jpg)

优化后点击商品价格改价：

![](https://ae01.alicdn.com/kf/Hd8eb006aab414041aea83683efe4cdaaO.jpg)

## 2.2 优化工作

- 首先在手机端显示缩率图和价格。隐藏查看更多。

核心代码：替换原有的价格显示样式，并添加一个js控制的弹窗。

```php+HTML
<td >
								<button class="am-btn am-btn-success price-changed am-btn-xs am-radius" data-id="{{$v.id}}" data-url="{{:MyUrl('admin/goods/pricechanged')}}" id="doc-prompt-toggle{{$v.id}}" >{{$v.price}}</button>
								<div class="am-modal am-modal-prompt" tabindex="-1" id="my-prompt{{$v.id}}">
								  <div class="am-modal-dialog">
								    <div class="am-modal-hd">{{$v.title}}</div>
								    <div class="am-modal-bd">
								      <img src="{{$v['images']}}" class="am-radius" style="width:100%"/>
								      <input type="number" class="am-modal-prompt-input"  pattern="^([0-9]{1}\d{0,6})(\.\d{1,2})?$" data-validation-message="请填写有效的销售金额" placeholder="{{$v.price}}">元
								    </div>
								    <div class="am-modal-footer">
								      <span class="am-modal-btn" data-am-modal-cancel>取消</span>
								      <span class="am-modal-btn" data-am-modal-confirm>提交</span>
								    </div>
								  </div>
								</div>
								
								{{if !empty($v['original_price']) and $v['original_price'] gt 0}}
									<br /><span class="am-badge am-radius">原价 {{$v.original_price}}</span>
								{{/if}}
							</td>
```

完整`index.html`代码链接：https://github.com/xieqifei/shopxo/blob/master/application/admin/view/default/goods/index.html

将此链接中的代码下载下来，替换原后台商品管理的html代码。替换`你的网站根目录/application/admin/view/default/goods/index.html`。

- 修改js文件，实现点击价格，弹出对话框，提交发送post请求修改数据库

```javascript
	/**
	 * [doc-prompt-toggle 价格修改弹窗]
	 * @author   Qifei
	 * @blog     https://sci.ci/
	 * @version  0.0.1
	 * @datetime 2020-05-25T14:22:39+0800
	 * @param    {[int] 	[data-id] 	[数据id]}
	 * @param    {[int] 	[data-field][数据库内容]}
	 * @param    {[string] 	[data-url] 	[请求地址]}
	 */
	 $(document).on('click','.price-changed', function() {
	 	var tag = $(this);
	 	var id = tag.attr('data-id');
	 	var $my_prompt='#my-prompt'+id;
	 	var url = tag.attr('data-url');
		var field = tag.attr('data-field') || '';
	    $($my_prompt).modal({
	      relatedElement: this,
	      onConfirm: function(data) {
            //获取input标签中的value值
	      	price=data['data'];
	      	if(id == undefined || url == undefined)
			{
				Prompt('参数配置有误');
				return false;
			}
			// 请求更新数据
			$.ajax({
				url:url,
				type:'POST',
				dataType:"json",
				timeout:tag.attr('data-timeout') || 30000,
				data:{"id":id, "price":price, "field":field},
				success:function(result)
				{
					if(result.code == 0)
					{
						Prompt(result.msg, 'success');
						// 成功则更新价格
						tag.html(price);
					} else {
						Prompt(result.msg);
					}
				},
				error:function(xhr, type)
				{
					Prompt('网络异常出错');
				}
			});
	      },
	      onCancel: function() {
	      }
	    });
	  });
  
```

> 以上代码加入到common.js文件中，或者下载我的commonjs完全替换这个文件

替换`你的网站/public/static/common/js/common.js`

完整代码：https://github.com/xieqifei/shopxo/blob/master/public/static/common/js/common.js

- 在Goods控制器里新建一个控制器函数PriceChanged()。实现接收POST数据。

在`application/admin/controller/Goods.php`里添加下面的代码。或者完全替换为我的`Goods.php`。

```php
	/**
	 * [PriceChanged 修改商品价格]
	 * @author   Qifei
	 * @blog     https://sci.ci/
	 * @version  0.0.1
	 * @datetime 2020-05-25T22:23:06+0800
	 */
	 public function PriceChanged()
	{
		// 是否ajax
		if(!IS_AJAX)
		{
			return $this->error('非法访问');
		}
		$filename="a.txt";
		$handle=fopen($filename,"a+");
		$str=fwrite($handle,"test\n");
		 fclose($handle);
		// 开始操作
		$params = input('post.');
		$params['admin'] = $this->admin;
		$params['field'] = 'price';
         //调用商品管理服务，更新数据库。
		return GoodsService::GoodsPriceUpdate($params);
	}
```

替换`你的网站/application/admin/controller/Goods.php`

完整的Goods.php代码：https://github.com/xieqifei/shopxo/blob/master/application/admin/controller/Goods.php

- 在商品后台服务中添加更新价格的服务函数

将以下代码添加到`你的网站/application/service/GoodsService.php`,或者下载我的`GoodsService.php`文件，完全替换它。

```php
/**
     * 修改商品价格
     * @author   Qifei
     * @blog    https://sci.ci/
     * @version 1.0.0
     * @date    2020-05-25
     * @desc    description
     * @param   [int]          $goods_id [商品id]
     */
    public static function GoodsPriceUpdate($params = [])
    {
        // 请求参数
        $p = [
            [
                'checked_type'      => 'empty',
                'key_name'          => 'id',
                'error_msg'         => '操作id有误',
            ],
            [
                'checked_type'      => 'empty',
                'key_name'          => 'field',
                'error_msg'         => '未指定操作字段',
            ],
            [//价格在0到5000这个区间
                'checked_type'      => 'between',
                'key_name'          => 'price',
                'checked_data'      => [0,5000],
                'error_msg'         => '价格不能小于0，大于5000',
            ],
        ];
        $ret = ParamsChecked($params, $p);
        if($ret !== true)
        {
            return DataReturn($ret, -1);
        }
		
        // 数据更新
        if(Db::name('Goods')->where(['id'=>intval($params['id'])])->update([$params['field']=>number_format($params['price'],2),'min_price'=>number_format($params['price'],2),'max_price'=>number_format($params['price'],2), 'upd_time'=>time()])&&Db::name('GoodsSpecBase')->where(['goods_id'=>intval($params['id'])])->update([$params['field']=>number_format($params['price'],2)]))
        {
            return DataReturn('操作成功');
        }
        return DataReturn('操作失败', -100);
    }
```

完整`GoodsService.php`地址：https://github.com/xieqifei/shopxo/blob/master/application/service/GoodsService.php

替换`你的网站/application/service/GoodsService.php`

- 最后，你需要给新的控制器方法添加访问数据库权限。

打开ShopXO网站后台`你的网站网址/admin.php`

![](https://ae01.alicdn.com/kf/H293d34dbf9a147c592d74b4954d64654G.jpg)

添加权限：![](https://ae01.alicdn.com/kf/He9b4b848c75644fcbdc2c2d960cb52d5Y.jpg)

点击保存。至此商品后台优化已经完成。

# 3：WX小程序优化

## 3.1 优化说明

> 优化内容：
>
> 1. 修改首页导航栏上传图片，显示太小
> 2. 修改分类无法查看一级分类。

修改前后对比

![](https://ae01.alicdn.com/kf/H7290adc40303410fbb2bb29573e26349o.jpg)

![](https://ae01.alicdn.com/kf/H1f69c3e41d8b45c6ae974c37aa3c0e08R.jpg)

## 3.2 优化工作

- 导航栏修改

1. 打开小程序源代码，找到`components`->`icon-nav`

替换`icon-nav.wxml`内容为：

```html
<view wx:if="{{propData.length > 0}}">
  <view class="data-list">
    <view class="items" wx:for="{{propData}}" wx:key="key">
      <view wx:if="{{item.bg_color==='#FFFFFF'}}" class="items-content" data-value="{{item.event_value}}" data-type="{{item.event_type}}" bindtap="navigation_event" style="background-color:#fff;padding:0;height:110rpx;width:110rpx;border-radius:0%">
        <image src="{{item.images_url}}" mode="aspectFit" style="width:110rpx;height:110rpx"/>
      </view>
      <view wx:else class="items-content" data-value="{{item.event_value}}" data-type="{{item.event_type}}" bindtap="navigation_event" style="background-color:{{item.bg_color}};">
        <image src="{{item.images_url}}" mode="aspectFit" style=""/>
      </view>
      <view class="title">{{item.name}}</view>
    </view>
  </view>
</view>
```

> 代码逻辑：当检测到导航栏背景颜色为白色时，设置图片上一层View标签的内边距padding 为0（原为20rpx）高宽设置为110rpx。并且将图片高宽均增加40rpx（原为70rpx），弥补内边距。

2. 修改`icon-nav.wxss`文件

```css
.data-list .items image {
  width: 60rpx ;
  height: 60rpx ;
  margin-top: 5rpx;
}
```

将原来的高宽后面的`!important`去掉，为了使步骤1中内联的高宽样式生效。

最后，特别注意：你在网站后台修改导航栏图片时，必须设置导航栏背景色为白色。不设置或者设置为其他颜色，则采用默认的样式。此时图片会比较小。

修改导航栏：`你的网站/admin.php`->`手机端管理`->`首页导航`,修改导航栏

![](https://ae01.alicdn.com/kf/H74bc60750cb04428817d42273a0f89fa6.jpg)

确保颜色代码为`#FFFFFF`

![](https://ae01.alicdn.com/kf/H2adb7476accc4be3b43acb4517f560d0h.jpg)

- 添加分类`全部`选项

完全替换小程序源代码中`pages`->`goods-category`->`goods-category.wxml`文件

```php+HTML
<view class='left-nav'>
  <block wx:for="{{data_list}}" wx:key="key">
    <view class='items {{item.active || ""}}' data-index="{{index}}" bindtap='nav_event'>
      <text>{{item.name}}</text>
    </view>
  </block>
</view>
<view class='right-content bg-white'>
  <view class="content-items" data-value="{{data_first_class['id']}}" bindtap="category_event">
        <view class="text single-text">全部</view>
  </view>
  <block wx:if="{{data_content.length > 0}}">
    <block wx:for="{{data_content}}" wx:key="keys" wx:for-item="v">
      <view class="content-items" data-value="{{v.id}}" bindtap="category_event">
        <image wx:if="{{(v.icon || null) != null}}" src="{{v.icon}}" mode="aspectFit" class="icon" />
        <view class="text single-text">{{v.name}}</view>
      </view>
    </block>
  </block>
</view>

<view wx:if="{{data_list.length == 0 && data_list_loding_status != 0}}">
  <import src="/pages/common/nodata.wxml" />
  <template is="nodata" data="{{status: data_list_loding_status}}">
  </template>
</view>
```

修改`pages`->`goods-category`->`goods-category.js`中部分代码

```js
// 导航事件
  nav_event(e) {
    var index = e.currentTarget.dataset.index;
    var temp_data = this.data.data_list;
    for(var i in temp_data)
    {
      temp_data[i]['active'] = (index == i) ? 'nav-active' : '';
    }
    this.setData({
      data_list: temp_data,
      data_first_class: temp_data[index],
      data_content: temp_data[index]['items'],
    });
  },
```

编译运行，即修改完毕。

> 有个小bug，打开分类页面，第一个一级分类的“全部”，在第一次打开时，会指向全部商品，而不是一级分类的商品。切换到第二个一级分类，再回来，此时点击“全部”，就变正常了。。后续再修复吧。

# 4：新订单通知推送

新订单推送，是为了方便商家，接收各个端下单消息。通常，通常都是使用短信和邮箱方式来接收消息，但是邮箱接收效率太低。短信又需要收费。

这里我们使用两种及时的推送方式，第一种，飞鹅打印机。入住美团、饿了么的商家，通常都会购买这个打印机，因为他是全平台的，而且可以调用它的API接口，自定义打印内容。

第二种，微信公众号推送。当网站后台接收到新订单，即通过微信公众号的模板消息功能，向指定微信推送新订单消息。

当然如果你具备一些编程知识，还可以自己写个程序，向下单的用户推送订单的进展。（开发一个插件，调用订单的相关钩子。）

![](https://ae01.alicdn.com/kf/Hbf03630cbb7b499e946293ebe107df1fU.jpg)

为了减少工作量，消息通知，是在原有插件`新订单提醒`基础上进行的修改。新的插件具有了除邮箱和短信以外的公众号和飞鹅打印功能。

![](https://ae01.alicdn.com/kf/H47729d1cdad342dbbf295c7e53404a9cQ.jpg)

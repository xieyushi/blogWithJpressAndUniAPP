<template>
	<view class="page-container">
			<!-- #ifdef MP-WEIXIN -->
		<view class="page-header">
			<button class="menu-icon" @tap="showRightDrawer">
				<uni-icon type="bars" color="#666666" size="22"></uni-icon>
			</button>
			<button class="share-btn" open-type="share">
				<uni-icon type="redo" size="26" color="#999999"></uni-icon>
			</button>
		</view>
			<!-- #endif -->
		<uni-drawer :visible="rightDrawerVisible" mode="right" @close="closeRightDrawer">
			<view class="drawer-header">
				<view >文章分类</view>
			</view>
			<view class="uni-list uni-common-mt">
				<view class="uni-list-cell" hover-class="uni-list-cell-hover">
					<view class="uni-list-cell-navigate uni-navigate-right" @tap="loadCategoryArticle">
						{{'全部('+totalRow+')'}}
					</view>
				</view>
				<view class="uni-list-cell" hover-class="uni-list-cell-hover" :data-categoryid="category.id"
				 v-for="(category,index) in categories" :key="index" @tap="loadCategoryArticle">
					<view class="uni-list-cell-navigate uni-navigate-right">
						{{category.title+'('+category.count+')'}}
					</view>
				</view>
			</view>
		</uni-drawer>
		<navigator class="uni-card" :url="'../detail/detail?id='+item.id" @tap="addViewCount" :data-index="index" 
		 v-for="(item,index) in articles" :key="index">
			<view class="uni-card-header">{{item.title}}</view>
			<view class="uni-card-content">
				<view class="uni-card-content-inner">
					<view class="article-content">
						{{item.text}}
					</view>
				</view>
			</view>
			<view class="uni-card-footer">
				<view class="article-create-time">
					{{item.created}}
				</view>
				<view class="article-icon">
					<uni-icon type="eye" size="18" color="#999999"></uni-icon>
					<view class="article-count">
						{{item.view_count}}
					</view>
				</view>
				<view class="article-icon">
					<uni-icon type="chat" size="18" color="#999999"></uni-icon>
					<view class="article-count">
						{{item.comment_count}}
					</view>
				</view>
			</view>
		</navigator>
		<view class="loadMore" v-if="showLoadMore">{{loadMoreText}}</view>
	</view>
</template>

<script>
	import uniDrawer from '../../components/uni-drawer.vue';
	import uniIcon from '../../components/uni-icon.vue'
	import jpress from "../../common/jpress.js"
	export default {
		data() {
			return {
				articles: [],
				categories: [],
				slides: [],
				indicatorDots: false,
				autoplay: false,
				interval: 5000,
				duration: 1000,
				loadMoreText: "加载更多...",
				showLoadMore: false,
				listCategoryId: "",
				listPageNumber: 1,
				noMoreArticle:false,
				rightDrawerVisible: false,
				totalRow:0,
				shareText: '你的好友分享给你一篇博客',
				image: '../../static/logo.png',
				shareType:0,
				providerList: [],
				shareData:{},
				pageLength:0
			};
		},
		components: {
			uniIcon,
			uniDrawer
		},
		onLoad: function(options) {
			uni.showLoading();
			this.loadCategories();
			this.loadArticleList();
			console.log(jpress);
		},
		onShareAppMessage() {
			return {
				title: '你的好友分享给你一个博客小程序',
				path: 'pages/index/index'
			}
		},
		onPullDownRefresh(){
			this.showLoadMore = true;
			this.listPageNumber = 1;
			this.noMoreArticle = false;
			this.loadArticleList(true);
		},
		onReachBottom() {
			if(this.noMoreArticle){
				this.showLoadMore = true;
				this.loadMoreText = "没有更多数据了!"
			}else{
				this.showLoadMore = true;
				this.listPageNumber++;
				this.loadArticleList();
			}
		},
		onNavigationBarButtonTap(obj){
			console.log(obj)
			if(obj.index == 1){
				this.shareBlog();
			}
			if(obj.index == 0){
				this.rightDrawerVisible = !this.rightDrawerVisible;
			}
		},
		methods:{
			addViewCount(e){
				//不想改jpress.所以这请求一下页面,刷新观看记数.
				uni.request({
					url:jpress.config.host+'/article/'+this.articles[e.currentTarget.dataset.index].id
				})
			},
			loadCategoryArticle(e){
				this.showLoadMore = true;
				this.listPageNumber = 1;
				this.noMoreArticle = false;
				this.listCategoryId = e.currentTarget.dataset.categoryid?e.currentTarget.dataset.categoryid:'';
				this.loadArticleList(true);
			},
			closeRightDrawer() {
				this.rightDrawerVisible = false;
			},
			showRightDrawer() {
				this.rightDrawerVisible = true;
			},
			loadCategories(){
				jpress.getArticleCategories("category")
				.then(data=>{
					this.categories = data.categories;
				})
			},
			loadArticleList(newLoad) {
				if(this.rightDrawerVisible){
					this.rightDrawerVisible = !this.rightDrawerVisible;
				}
				jpress.getArticlePage({
						categoryId: this.listCategoryId,
						page: this.listPageNumber,
					})
					.then(data => {
						if(this.totalRow == 0){
							this.totalRow = data.page.totalRow;
						}
						if(data.page.list.length>0){
							if(data.page.list <= 10){
								showLoadMore = false;
							}
							if(newLoad){
								this.articles = data.page.list;
								uni.stopPullDownRefresh();
							}else{
								this.articles =this.articles.concat(data.page.list);
							}
						}else{
							this.loadMoreText = "没有更多数据了!"
							this.noMoreArticle = true;
						}
						uni.hideLoading();
					})
			}
			//#ifdef APP-PLUS
			,
			compress(){//压缩图片 图文分享要求分享图片大小不能超过20Kb
				console.log("开始压缩");
				let img = this.image;
				return new Promise((res) => {
					var localPath = plus.io.convertAbsoluteFileSystem(img.replace('file://', ''));
					console.log('after' + localPath);
					// 压缩size
					plus.io.resolveLocalFileSystemURL(localPath, (entry) => {
						entry.file((file) => {// 可通过entry对象操作图片 
							console.log("getFile:" + JSON.stringify(file));
							if(file.size > 20480) {// 压缩后size 大于20Kb
								plus.zip.compressImage({
									src: img,
									dst: img.replace('.jpg', '2222.jpg').replace('.JPG', '2222.JPG'),
									width: '10%',
									height: '10%',
									quality: 1,
									overwrite: true
								}, (event) => {
									console.log('success zip****' + event.size);
									let newImg = img.replace('.jpg', '2222.jpg').replace('.JPG', '2222.JPG');
									res(newImg);
								}, function(error) {
									uni.showModal({
										content:"分享图片太大,需要请重新选择图片!",
										showCancel:false
									})
								});
							}
						});
					}, (e) => {
						console.log("Resolve file URL failed: " + e.message);
						uni.showModal({
							content:"分享图片太大,需要请重新选择图片!",
							showCancel:false
						})
					});
				})
			},
			shareBlog(){
				uni.getProvider({
					service: "share",
					success: (e) => {
						console.log("success", e);
						let data = []
						for (let i = 0; i < e.provider.length; i++) {
							switch (e.provider[i]) {
								case 'weixin':
									data.push({
										name: '分享到微信好友',
										id: 'weixin',
										sort:0
									})
									data.push({
										name: '分享到微信朋友圈',
										id: 'weixin',
										type:'WXSenceTimeline',
										sort:1
									})
									break;
								case 'sinaweibo':
									data.push({
										name: '分享到新浪微博',
										id: 'sinaweibo',
										sort:2
									})
									break;
								case 'qq':
									data.push({
										name: '分享到QQ',
										id: 'qq',
										sort:3
									})
									break;
								default:
									break;
							}
						}
						data = data.sort((x,y) => {
						return x.sort - y.sort
					});
						this.providerList = data;
						let stringarray = [];
						for (var i = 0; i < data.length; i++) {
							stringarray.push(data[i].name);
						}
						uni.showActionSheet({
							itemList: stringarray,
							success: ress => {
								for (var i = 0; i < data.length; i++) {
									if(i == ress.tapIndex){
										this.shareData = data[i];
									}
								}
								this.share();
							},
							fail: ress => {
								uni.showToast({
									title: '欢迎您再来分享!',
									icon: 'none',
									mask: true
								});
							}
						});
					},
					fail: (e) => {
						console.log("获取登录通道失败", e);
						uni.showModal({
							content:"获取登录通道失败",
							showCancel:false
						})
					}
				});
				
			},
			async share(){
				var e = this.shareData;
				
				
				let shareOPtions = {
					provider: e.id,
					scene: e.type && e.type === 'WXSenceTimeline' ? 'WXSenceTimeline' : "WXSceneSession", //WXSceneSession”分享到聊天界面，“WXSenceTimeline”分享到朋友圈，“WXSceneFavorite”分享到微信收藏     
					type: this.shareType,
					success: (e) => {
						console.log('success'+JSON.stringify(e));
					},
					fail: (e) => {
						console.log('fail'+JSON.stringify(e));
					},
					complete:function(e){
						if(e.errMsg == 'share:ok'){
							uni.showModal({
								content:"谢谢分享!",
								showCancel:false
							})
						}else{
							uni.showModal({
								content:"哎呀?为毛不分享?",
								showCancel:false
							})
						}
					}
				}
				
				shareOPtions.summary = this.shareText;
				shareOPtions.imageUrl = this.image;
				shareOPtions.title = '你的好友分享给你一个博客'
				shareOPtions.href = "https://blog.coder666.cn";
				uni.share(shareOPtions);
			}
			//#endif
			
		}
	}
</script>

<style>
	page{
		background: #F5F5F5;
	}
	.share-btn{
		background:#ffffff;
		align-self: flex-end;
		margin-right: 20upx;
	}
	.menu-icon{
		background:#ffffff;
		align-self: flex-start;
		margin-left: 20upx;
	}
	.share-btn::after{ 
		border: none; 
	}
	.menu-icon::after{ 
		border: none; 
	}
	.page-header{
		height: 100upx;
		line-height: 100upx;
		position: sticky;
		top: 0;
		width: 750upx;
		display:flex;
		flex-direction:row;
		align-items:center;
		background:#ffffff;
		z-index: 100;
	}
	.drawer-header{
		margin-top: 50upx;
		height: 100upx;
		display: flex;
		flex-direction: column;
		flex-wrap: nowrap;
		align-items: center;
		justify-content: flex-start;
	}
	.uni-icon-close{
		align-self: flex-end;
	}
	.uni-card{
		width: 670upx;
		margin: 20upx auto;
	}
	.page-container{
		width: 750upx;
		margin: 0 auto;
		display: flex;
		flex-direction: row;
		flex-wrap: wrap;
	}
	.article-container{
		width: 710upx;
		display: flex;
		flex-direction: column;
		background: #ffffff;
		margin-top:30upx;
	}
	.article-item{
		width: 710upx;
		height: 230upx;
		display: flex;
		flex-direction: row;
		flex-wrap: wrap;
		justify-content: center;
		align-items: center;
	}
	.article-title{
		width: 670upx;
		margin-top: 20upx;
		overflow : hidden;
		text-overflow: ellipsis;
		display: -webkit-box;
		-webkit-line-clamp: 1;
		-webkit-box-orient: vertical;
		color: #999999;
		font-size: 36upx;
	}
	.article-content{
		overflow : hidden;
		text-overflow: ellipsis;
		display: -webkit-box;
		-webkit-line-clamp: 2;
		-webkit-box-orient: vertical;
		font-size: 28upx;
		color:#999999;
		width:610upx;
	}
	.article-icon-container{
		width: 670upx;
		align-items: flex-end;
		
		height: 40upx;
		margin-bottom: 5upx;
		display: flex;
		flex-direction: row;
		justify-content: flex-end;
	}
	.article-icon{
		width: 160upx;
		font-size: 28upx;
		height: 40upx;
		color: #999999;
	}
	.article-count{
		margin-left: 20upx;
		height: 40upx;
		line-height: 40upx;
		overflow: hidden;
	}
	.article-create-time{
		height: 40upx;
		line-height: 40upx;
		overflow: hidden;
		font-size: 32upx;
		width: 200upx;
		margin-right: 150upx;
	}
	.loadMore {
		text-align: center;
		width: 710upx;
		height: 100upx;
		line-height: 100upx;
		display: block;
		font-size: 32upx;
		color: #B2B2B2;
	}
</style>

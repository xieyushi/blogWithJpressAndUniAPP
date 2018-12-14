<template>
	<view class="page-container">
		<view class="article-header">
			<button v-if="pageLength == 1" class="home-btn" @tap="homePage">
				<uni-icon type="home" size="26" color="#999999"></uni-icon>
			</button>
			<button v-else class="home-btn-hidden">
				<uni-icon type="home" size="26" color="#999999"></uni-icon>
			</button>
			<view class='details-title'>
				{{article.title}}
			</view>
			<!-- #ifdef MP-WEIXIN -->
			<button class="share-btn" open-type="share">
				<uni-icon type="redo" size="26" color="#999999"></uni-icon>
			</button>
			<!-- #endif -->
		</view>
		<wxParse :content="article.content" @preview="preview" @navigate="navigate" />
		<view class="newcomment-container" v-if="loadingData">
			<view class="comment-title">发起评论</view>
			<textarea v-model="commentText" class="newcomment-textarea" auto-height />
			<!-- #ifdef MP-WEIXIN -->
			<button class="comment-submit" open-type="getUserInfo" @tap="getUserInfo">发送</button>
			<!-- <button class="comment-submit" @tap="postComment">发送</button> -->
			<!-- #endif -->
			<!-- #ifdef APP-PLUS -->
			<!-- <button class="comment-submit" open-type="getUserInfo" @tap="getUserInfo">发送</button> -->
			<button class="comment-submit" @tap="postComment">发送</button>
			<!-- #endif -->
		</view>
		<view class="oldcomment-container" v-if="loadingData&&comments.length>0">
			<view class="comment-title">最新评论</view>
			<view class="comment-item" v-for="(comment,index) in comments" :key="index">
				<view class="oldcomment-title">
					<view class="oldcomment-nickName">
						{{comment.user?comment.user.nickname:'某人'}}
					</view>
					<view class="oldcomment-commentTime">
						{{comment.created}}
					</view>
				</view>
				<view class="oldcomment-content">
					{{comment.text}}
				</view>
			</view>
		</view>
		<view class="no-comments" v-else-if="loadingData&&comments.length==0">
			暂时还没有评论,沙发等你来!
		</view>
		<view class="loadMore" v-if="showLoadMore && comments.length>0">{{loadMoreText}}</view>
	</view>
</template>

<script>
	//本示例引用组件为三方的mpvue-wxparse，该组件为开源mit协议，如有新需求可自行改动完成或联系原作者
	import jpress from "../../common/jpress.js"
	import marked from '../../components/marked'
	import wxParse from '../../components/mpvue-wxparse/src/wxParse.vue'
	import uniIcon from '../../components/uni-icon.vue'
	//真实业务开发时mdcontend应改为从网络获取，本演示写死在本地
	var mdcontend = "";

	export default {
		components: {
			wxParse,
			uniIcon
		},
		data() {
			return {
				article : {},
				loadMoreText: "加载更多评论...",
				showLoadMore: false,
				relatedArticles:[],
				comments:[],
				commentContent:'',
				loadingData:false,
				copyright: jpress.globalData.copyright,
				commentIndex:1,
				noMoreComment:false,
				commentText:'',
				title: '',
				shareText: '你的好友分享给你一篇博客',
				href:"",
				image: '../../static/logo.png',
				shareType:0,
				providerList: [],
				shareData:{},
				pageLength:0
			}
		},
		onNavigationBarButtonTap(obj){
			this.shareBlog();
		},
		onShareAppMessage() {
			return {
				title: '你的好友分享给你一篇博客',
				path: 'pages/detail/detail?id='+this.article.id
			}
		},
		onReachBottom() {
			if(this.noMoreComment){
				this.showLoadMore = true;
				this.loadMoreText = "没有更多评论了!"
			}else{
				this.showLoadMore = true;
				this.commentIndex++;
				this.loadComment(this.commentIndex);
			}
		},
		onLoad: function(options) {
			//加载文章
			jpress.getArticle(options.id)
				.then(data=>{
					this.article = data.article;
					this.article.content = marked(this.article.content);
					this.loadComment(this.commentIndex);
					this.loadingData = true;
				})
				let pages = getCurrentPages();
				this.pageLength = pages.length;
		},
		methods: {
			homePage(){
				uni.redirectTo({
					url: '../index/index'
				});
			},
			preview(src, e) {
				// do something
			},
			navigate(href, e) {
				// 如允许点击超链接跳转，则应该打开一个新页面，并传入href，由新页面内嵌webview组件负责显示该链接内容
				uni.showModal({
					content: "点击链接为：" + href,
					showCancel: false
				})
			},
			getUserInfo(e){
				if(this.commentText){
					
				}else{
					uni.showToast({
						title: '请填写评论内容',
						icon: 'none',
						mask: true
					});
					return;
				}
				uni.login({
				success:res=> {
					if (res.code) {
					jpress.wxLogin(res.code);
					uni.getUserInfo({
					provider: 'weixin',
					success: infoRes => {
						console.log(infoRes)
						jpress.wxGetUserInfo(infoRes, res =>{
							if(res){
								jpress.doPostComment({
									articleId:this.article.id
								},
								this.commentText
								)
								.then(data=>{
									this.comments = [data.comment].concat(this.comments);
									uni.showToast({
										title: '评论成功',
										icon: 'none',
										mask: true
									});
									this.commentText = '';
								})
							}
						});
					}
					});
					}
				}
				})
				
			},
			//APP的话.直接评论,不用登录.因为登录授权还没下来.只能匿名评论...
			postComment(e) {
				uni.request({
					url:jpress.config.host+'/article/postComment',
					method:'GET',
					dataType:'application/x-www-form-urlencoded',
					data:{
						articleId:this.article.id,
						pid:'',
						content:this.commentText
						},
					success:res =>{
						console.log(res);
						let resdata = JSON.parse(res.data);
						if(resdata.state == 'ok'){
							this.comments = [resdata.comment].concat(this.comments);
							uni.showToast({
								title: '评论成功',
								icon: 'none',
								mask: true
							});
							this.commentText = '';
						}
					}
				})

			},
			loadComment(page) {

				jpress.getCommentPage({
						articleId: this.article.id,
						page: page
					})
					.then(data => {
						if(data.page.list.length>0){
							this.comments =this.comments.concat(data.page.list);
						}else{
							this.noMoreComment = true;
							this.loadMoreText = "没有更多评论了!"
						}
					})
			},
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
				
				if(!this.article.title && (this.shareType === 1 || this.shareType === 0)){
					uni.showModal({
						content:"分享内容不能为空",
						showCancel:false
					})
					return;
				}
				
				if(!this.image && (this.shareType === 2 || this.shareType === 0)){
					uni.showModal({
						content:"分享图片不能为空",
						showCancel:false
					})
					return;
				}
				
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
				shareOPtions.title = this.article.title;
				shareOPtions.href = jpress.config.host+"/article/"+this.article.id;
				
				if(shareOPtions.type === 0 && plus.os.name === 'iOS'){//如果是图文分享，且是ios平台，则压缩图片 
					shareOPtions.imageUrl = await this.compress();
				}
				uni.share(shareOPtions);
			}

		}

	}
</script>

<style>
	@import url("../../components/mpvue-wxparse/src/wxParse.css");
	.page-container{
		display: flex;
		flex-direction: column;
		flex-wrap: wrap;
		width: 670upx;
		margin: 0 auto;
	}
	view{
		display: block !important;
		text-align: justify;
	}
	.article-header{
		width: 750upx;
		margin-left: -40upx;
		position: sticky;
		padding-top: 30upx;
		padding-bottom: 15upx;
		margin-bottom: 15upx;
		margin-top: 30upx;
		top:-1upx;
		background-color: #FFFFFF;
		display: flex !important;
		flex-direction: row;
		flex-wrap: nowrap;
		align-items: baseline;
		border-bottom: 2upx solid #F2F2F2;
	}
	.details-title{
		width: 550upx;
		font-size: 36upx;
		text-align: center;
	}
	.share-btn{
		width: 60upx;
		height: 60upx;
		background: #FFFFFF;
		margin: 0;
		padding: 0;
		margin-right: 40upx;
	}
	.share-btn::after{
		border: none;
	}
	.home-btn{
		width: 60upx;
		height: 60upx;
		background: #FFFFFF;
		margin: 0;
		margin-left: 40upx;
		padding: 0;
	}
	.home-btn-hidden{
		visibility: hidden;
		width: 60upx;
		height: 60upx;
		background: #FFFFFF;
		margin: 0;
		margin-left: 40upx;
		padding: 0;
	}
	.home-btn::after{
		border: none;
	}
	.newcomment-container{
		margin-bottom: 200upx;
		width: 670upx;
		margin-top: 200upx;
	}
	.newcomment-textarea{
		background: #F7F7F7;
		min-height: 200upx;
		border-radius: 10upx;
		width: 670upx;
		color: #999999;
		font-size: 28upx;
	}
	.comment-title{
		border-left: 4upx solid #4CD964;
		height: 40upx;
		font-size: 36upx;
		line-height: 40upx;
		padding-left: 10upx;
		margin-bottom: 20upx;
	}
	.comment-submit{
		margin-top: 20upx;
		width: 200upx;
		height: 80upx;
		font-size: 32upx;
		line-height: 80upx;
		text-align: center;
		background: #BBBBBB;
		color: #FFFFFF;
		float: right;
		border-radius: 10upx;
	}
	.oldcomment-textarea{
		background: #F7F7F7;
		min-height: 200upx;
		border-radius: 10upx;
		width: 670upx;
	}
	.oldcomment-container{
		width: 670upx;
		display: flex;
		flex-direction: column;
		border-radius: 20upx;
		color: #BBBBBB;
		font-size: 32upx;
	}
	.oldcomment-title{
		display: flex !important;
		flex-direction: row;
		flex-wrap: nowrap;
		height: 60upx;
		line-height: 60upx;
	}
	.oldcomment-nickName{
		width: 300upx;
	}
	.oldcomment-commentTime{
		width: 370upx;
	}
	.oldcomment-content{
		color: #999999;
	}
	.comment-item{
		margin-top: 20upx;
		border-bottom: 2upx dashed #C8C7CC;
		padding-bottom: 30upx;
	}
	.loadMore {
		text-align: center;
		width: 710upx;
		height: 100upx;
		line-height: 100upx;
		display: block;
	}
	.no-comments{
		height: 100upx;
		line-height: 100upx;
		font-size: 32upx;
		color: #B2B2B2;
		text-align: center;
	}
</style>

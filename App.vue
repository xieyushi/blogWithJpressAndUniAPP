<script>
	import jpress from "./common/jpress.js"
	export default {
		onLaunch: function () {
			//jpress 初始化
			   jpress.init({
			     host:'#@#jpressAPI配置#@#',
			     app_id:'#@#小程序appid#@#',
			     app_secret:'#@#jpress接口加密串,注意不是小程序的appsecret,只是一个加密串,可以是任意的#@#'
			   }) 
			    jpress.getAppName()
			      .then(data => {
			        if (data.value) {
								uni.setStorageSync('appName',data.value);
			        }
			      })
			
			
			    jpress.getCopyright()
			      .then(data => {
			        if (data.value) {
								uni.setStorageSync('copyright',data.value);
			        }
			      })
		},
		onShow: function () {
		},
		onHide: function () {
		}
	}
</script>

<style>
	/*每个页面公共css */
	@import "./common/uni.css";
	page,
	view {
		display: flex;/* uni-app默认使用flex布局。因为flex布局有利于跨更多平台，尤其是采用原生渲染的平台。如不了解flex布局，请参考http://www.w3.org/TR/css3-flexbox/。若不需要flex布局可删除本行*/
	}
	
	
</style>

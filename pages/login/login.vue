<template>
	<view class="login_page">
		<view class="status_bar" :style="{ height: statusBarHeight }">
		    <!-- 这里是状态栏 -->
		</view>
		<view class="uni-padding-wrap uni-common-pb login_top">
			<view class="return_btn" @tap="goMain"><text class="uni-icon uni-icon-home"></text></view> <button class="b login_title">登录</button>
		</view>
		<view class="content">
			<block v-if="!initAddr">
				<view class="input-group">
					<view class="input-row border">
						<text class="title">接口地址：</text>
						<input-box ref="interfaceAddrIpt" type="text" :inputValue="interfaceAddr" v-model="interfaceAddr"></input-box>
					</view>
				</view>
				<view class="btn-row">
					<button type="warn" @tap="setInterfaceAddr">请先配置接口服务地址</button>
				</view>
			</block>
			<block v-else>
				<view class="input-group">
					<view class="input-row border">
						<text class="title">账号：</text>
						<input-box ref="input1" type="text" :verification="['isNull']" :verificationTip="['帐号不能为空']" class="input-box" clearable focus :inputValue="account" v-model="account" placeholder="请输入经销商编号或身份证号"></input-box>
					</view>
					<view class="input-row">
						<text class="title">密码：</text>
						<input-box ref="input2" type="password" :verification="['isNull','isChineseEnlishAndNumber']" :verificationTip="['密码不能为空','密码只能输入中文、数字和字母']" :inputValue="password" v-model="password" placeholder="请输入密码"></input-box>
					</view>
					<view class="input-row">
						<text class="title">验证码：</text>
						<input-box class="voli_code_ipt" ref="input3" type="text" :verification="['isNull']" :verificationTip="['验证码不能为空']" rightText="看不清？" rightClass="right_txt" @rightClick="resetVoliCode" maxLength="4" v-model="voliCode" placeholder="请输入验证码"></input-box>
						<view class="voli_code_img"><image style="width: 100%;" mode="widthFix" :src="voliCodeSrc" @click="resetVoliCode"></image></view>
					</view>
				</view>
				<view class="btn-row">
					<button type="warn" @tap="bindLogin">登录</button>
				</view>
				<!-- <view class="action-row">
					<navigator url="./reg">注册账号</navigator>
					<text class="split">|</text>
					<navigator url="./pwd">忘记密码</navigator>
				</view> -->
				<view class="oauth-row" v-if="hasProvider" v-bind:style="{top: positionTop + 'px'}">
					<view class="oauth-image" v-for="provider in providerList" :key="provider.value">
						<image :src="provider.image" @tap="oauth(provider.value)"></image>
					</view>
				</view>
			</block>
		</view>
	</view>
</template>
<script>
	var statusBarHeight = uni.getSystemInfoSync().statusBarHeight + 'px'
	import service from '../../service.js';
	import {mapState, mapMutations} from 'vuex';
	// http://ext.dcloud.net.cn/plugin?id=449
	import inputBox from '@/components/input-box/input-box';
	import util from '@/common/util.js';
	import voliCodeImg from '@/static/img/makecaptcha.jpg';
	export default {
		components: {
		    inputBox
		},
		computed: mapState(['sessionId', 'openid']),
		data() {
			return {
				initAddr: true,
				statusBarHeight: statusBarHeight,
				interfaceAddr: '',
				uuid: '',
				providerList: [],
				hasProvider: false,
				account: 'A0000002',
				password: '',
				voliCode: '',
				voliCodeSrc: '',
				positionTop: 0
			}
		},
		onLoad() {
			console.log('onLoad');
			this.getSessionId();
			// #ifdef APP-PLUS
			// plus.nativeUI.showWaiting('加载中……');
			// this.getSystemInfo();
			// #endif
		},
		onReady() {
			console.log('onReady');
			// #ifdef APP-PLUS
			// plus.nativeUI.closeWaiting();
			// #endif
			// this.initPosition();
			// this.initProvider();
			// #ifdef H5
			document.onkeydown = e => {  
			    //webview不需要兼容ie  
				if (e.keyCode === 13) {
					this.bindLogin();
				}
			}
			// #endif
		},
		mounted() {
			console.log('mounted');
			this.interfaceAddr = util.getBaseUrl();
			// this.$refs.interfaceAddrIpt.setValue(this.interfaceAddr);
			let sessionId = util.getStorageSync('session_id');
			// if (sessionId) {
			// 	this.autoLogin(sessionId);
			// }
			// console.log('sessionId: ', sessionId);
			this.getDeviceId();
		},
		methods: {
			...mapMutations(['login', 'setSessionId', 'setOpenid', 'setUserInfo']),
			autoLogin(sessionId) {
				let me = this;
				this.setOpenid(sessionId);
				// util.showLoading({
				// 	title: '自动登录中'
				// });
				console.log(this.interfaceAddr + 'json.rpc/webapi')
				uni.request({
					url: this.interfaceAddr + 'json.rpc/webapi',
					data: {
						jsonrpc: '2.0',
						method: 'SYS.UserDAL.GetDealerByToken',
						params: {},
						id: 1,
						tags: {
							usertoken: sessionId
						}
					},
					header: {},
					method: 'POST',
					dataType: 'json',
					responseType: 'text',
					success(res) {
						util.hideLoading();
						if (res.statusCode === 200) {
							if (res.data.hasOwnProperty('error')) {
								// util.showToast({
								// 	title: '登录已过期，请重新登录'
								// });
							} else if (res.data.hasOwnProperty('result')) {
								res.data.result = util.jsonReplace(res.data.result, 'null', '""');
								me.setUserInfo(res.data.result);
								me.toMain(res.data.result.DealerName);
							}
						}
					},
					fail(err) {
						util.hideLoading();
						console.log(err);
						if (err.hasOwnProperty('errMsg') && err.errMsg.indexOf('request:fail abort') !== -1) {
							util.dialog({
								title: '错误信息',
								content: '网络错误，请查看网络是否开启',
								showCancel: false
							});
						}
					}
				});
			},
			goMain() {
				util.goTab({
					url: '/pages/tabBar/index'
				});
			},
			initPosition() {
			    /**
			     * 使用 absolute 定位，并且设置 bottom 值进行定位。软键盘弹出时，底部会因为窗口变化而被顶上来。
			     * 反向使用 top 进行定位，可以避免此问题。
			     */
			    this.positionTop = uni.getSystemInfoSync().windowHeight - 100;
				console.log('initPosition: ', this.positionTop);
			},
			initProvider() {
			    const filters = ['weixin', 'qq', 'sinaweibo'];
			    uni.getProvider({
			        service: 'oauth',
			        success: res => {
						console.log('initProvider: ', res)
			            if (res.provider && res.provider.length) {
			                for (let i = 0; i < res.provider.length; i++) {
			                    if (~filters.indexOf(res.provider[i])) {
			                        this.providerList.push({
			                            value: res.provider[i],
			                            image: '../../static/img/' + res.provider[i] + '.png'
			                        });
			                    }
			                }
			                this.hasProvider = true;
			            }
			        },
			        fail: (err) => {
			            console.error('获取服务供应商失败：' + JSON.stringify(err));
			        }
			    });
			},
			oauth(value) {
			    uni.login({
			        provider: value,
			        success: (res) => {
			            uni.getUserInfo({
			                provider: value,
			                success: (infoRes) => {
			                    /**
			                     * 实际开发中，获取用户信息后，需要将信息上报至服务端。
			                     * 服务端可以用 userInfo.openId 作为用户的唯一标识新增或绑定用户信息。
			                     */
			                    this.toMain(infoRes.userInfo.nickName);
			                }
			            });
			        },
			        fail: (err) => {
			            console.error('授权登录失败：' + JSON.stringify(err));
			        }
			    });
			},
			toMain(userName) {
				this.login(userName);
				this.goMain();
				/**
				 * 强制登录时使用reLaunch方式跳转过来
				 * 返回首页也使用reLaunch方式
				 */
				// if (this.forcedLogin) {
				// 	uni.reLaunch({
				// 		url: '../main/main',
				// 	});
				// } else {
				// 	uni.navigateBack();
				// }
			},
			async getSessionId() {
				await util.ajax({
					method: 'SYS.UserDAL.GetSessionId'
				}).then(res => {
					// console.log('GetSessionId: ', res);
					this.setSessionId(res.data.result);
				});
				this.resetVoliCode();
			},
			resetVoliCode() {
				// console.log('验证码');
				// console.log('sessionId: ', this.sessionId);
				util.ajax({
					method: 'SYS.UserDAL.GetVerifyCodeBase64',
					tags: {
						usertoken: '',
						sessionid: this.sessionId
					}
				}).then(res => {
					util.hideLoading();
					// console.log('获取验证码： ', res);
					this.voliCodeSrc = 'data:image/jpeg;base64,' + res.data.result;
					
					// #ifdef APP-PLUS
					// plus.nativeUI.closeWaiting();
					// #endif
				});
				// util.ajax({
				// 	get: true,
				// 	responseType: 'arraybuffer',
				// 	method: 'images/verifycode',
				// }).then(res => {
				// 	console.log(res);
				// 	var buffer = new Buffer(res.data.byteLength);
				// 	var view = new Uint8Array(res.data);
				// 	for (var i = 0, len = buffer.length; i < len; ++i) {
				// 		buffer[i] = view[i];
				// 	}
				// 	this.voliCodeSrc = 'data:image/jpeg;base64,' + buffer.toString("base64");
				// });
			},
			setInterfaceAddr() {
				// #ifdef APP-PLUS
				// plus.nativeUI.showWaiting('加载中……');
				// #endif
				util.showLoading();
				util.setBaseUrl(this.interfaceAddr);
				this.initAddr = true;
				this.$nextTick(() => {
					this.$refs.input1.setValue(this.account);
					this.getSessionId();
				});
			},
			getDeviceId() {
				let me = this;
				try{
					plus.device.getInfo({
						success(e) {
							console.log('plus.device.getInfo: ', e);
							me.uuid = e.uuid;
						}
					});
				}catch(e){
					
				}
			},
			async bindLogin() {
				let me = this;
				console.log(this.$refs);
				if(!this.$refs.input1) {
					return;
				}
				const data = {
				    account: this.$refs.input1.getValue(),
				    password: this.$refs.input2.getValue(),
				    voliCode: this.$refs.input3.getValue()
				}
				if(this.$refs.input1.getValue() && this.$refs.input2.getValue() && this.$refs.input3.getValue()){
					util.showLoading();
					await util.ajax({
						method: 'SYS.UserDAL.Login',
						params: {
							UserName: data.account,
							LoginInfo: {
								Password: data.password,
								VerifyCode: data.voliCode,
								ClientType: 1,
								ClientVersion: '1.0.0.1',
								EncyptType: 0,
								DeviceId: this.uuid
							}
						},
						tags: {
							usertoken: '',
							sessionid: this.sessionId
						}
					}).then(res => {
						// console.log('登录返回： ', res);
						util.setStorageSync({
							key: 'session_id',
							data: res.data.result
						});
						this.setOpenid(res.data.result);
					});
					// console.log('openid: ', this.openid);
					await util.ajax({
						method: 'SYS.UserDAL.GetDealerByToken',
						tags: {
							usertoken: me.openid
						}
					}).then(res => {
						util.hideLoading();
						console.log('经销商信息：', res.data);
						res.data.result = util.jsonReplace(res.data.result, 'null', '""');
						me.setUserInfo(res.data.result);
						me.toMain(res.data.result.DealerName);
					});
					console.log('登录成功');
					// util.goTab({
					// 	url: '../tabBar/user?logined=true',
					// 	success: (e) => {
							// let page = getCurrentPages().pop();
							// console.log('page: ', page);
							// if (page == undefined || page == null) return;
							// page.login();
					// 	}
					// });
                } else {
					uni.showToast({
					    icon: 'none',
					    title: '请正确填写完整',
					    // #ifdef MP-WEIXIN
					    duration: 2000,
					    // #endif
					    // mask: true
					});
				}
			},
			getSystemInfo() {
				uni.getSystemInfo({
					success (res) {
						uni.showModal({
							title: '系统信息',
							content: 'model: ' + res.model + '\n'
									+ 'pixelRatio: ' + res.pixelRatio + '\n'
									+ 'windowWidth: ' + res.windowWidth + '\n'
									+ 'windowHeight: ' + res.windowHeight + '\n'
									+ 'language: ' + res.language + '\n'
									+ 'version: ' + res.version + '\n'
									+ 'platform: ' + res.platform + '\n'
									+ 'SDKVersion: ' + res.SDKVersion + '\n'
									+ 'system: ' + res.system + '\n'
									+ 'statusBarHeight: ' + res.statusBarHeight,
							success (resOperate) {
								if (resOperate.confirm) {
									console.log('用户点击确定');
								} else if (resOperate.cancel) {
									console.log('用户点击取消');
								}
							}
						});
					}
				});
			}
		}
	}
</script>

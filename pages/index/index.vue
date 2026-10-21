<template>
	<view class="screen">
		<view class="stage" :style="stageStyle">
			<!-- 顶栏 -->
			<view class="topbar">
				<view class="brand">
					<svg class="logo" viewBox="0 0 44 44">
						<path d="M22 3 L38 10 V22 C38 32 31 39 22 42 C13 39 6 32 6 22 V10 Z" fill="none"
							stroke="#38bdf8" stroke-width="1.6" opacity="0.9" />
						<path d="M22 3 L38 10 V22 C38 32 31 39 22 42 C13 39 6 32 6 22 V10 Z" fill="#0b2a3d"
							opacity="0.7" />
						<path d="M15 32 V21 a7 7 0 0 1 14 0 V32" fill="none" stroke="#7fe3ff" stroke-width="2" />
						<circle cx="22" cy="20" r="2.4" fill="#7fe3ff" />
					</svg>
					<view class="brand-txt">
						<view class="brand-line">
							<text class="brand-name">隧安卫</text>
							<text class="brand-en">TUNNEL · GUARD</text>
						</view>
						<text class="brand-sub">双无线隧域地下智能安防系统 · ESP-NOW + LoRa + WiFi 三级链路</text>
					</view>
				</view>

				<view class="steps">
					<view v-for="(s, i) in steps" :key="s.name" class="step"
						:class="{ on: state.step === i, done: state.step > i }" @click="go(i)">
						<text class="step-i">0{{ i + 1 }}</text>
						<text class="step-n">{{ s.short }}</text>
					</view>
				</view>

				<view class="topright">
					<view class="chip">
						<text class="c-k">节点</text>
						<text class="c-v">{{ onlineCount }}/3</text>
					</view>
					<view class="chip">
						<text class="c-k">心跳</text>
						<text class="c-v">{{ hbText }}s</text>
					</view>
					<view class="chip" :class="{ danger: alertCount > 0 }">
						<text class="c-k">告警</text>
						<text class="c-v">{{ alertCount }}</text>
					</view>
					<text class="clock">{{ clock }}</text>
				</view>
			</view>

			<!-- 主区 -->
			<view class="main">
				<!-- 左：三块板 -->
				<view class="col-left">
					<view class="board" v-for="c in cards" :key="c.key" :class="'b-' + c.tone">
						<view class="b-hd">
							<text class="b-ico">{{ c.icon }}</text>
							<view class="b-tt">
								<text class="b-name">{{ c.name }}</text>
								<text class="b-sub">{{ c.sub }}</text>
							</view>
							<view class="b-pill" :class="'p-' + c.tone">
								<text>{{ c.badge }}</text>
							</view>
						</view>

						<view class="b-rows">
							<view class="row" v-for="(r, ri) in c.rows" :key="ri">
								<text class="r-k">{{ r.k }}</text>
								<view class="r-mid">
									<view class="r-bar" v-if="r.pct !== null">
										<view class="r-fill" :class="'f-' + r.tone"
											:style="{ width: (r.pct < 2 ? 2 : r.pct) + '%' }"></view>
										<view class="r-tick" v-if="r.tick" :style="{ left: r.tick + '%' }"></view>
									</view>
									<text class="r-v" :class="'v-' + r.tone">{{ r.v }}<text class="r-u">{{ r.u }}</text></text>
								</view>
							</view>
						</view>

						<view class="b-ft">
							<text v-for="(f, fi) in c.foot" :key="fi" class="ft-i">{{ f }}</text>
						</view>
					</view>
				</view>

				<!-- 中：3D 场景 + 链路 -->
				<view class="col-mid">
					<view class="scene-card">
						<view class="panel-hd">
							<text class="panel-title">现场数字孪生 · 等轴测剖面</text>
							<view class="hd-right">
								<view class="mode-switch">
									<text class="ms" :class="{ on: mode === 'road' }" @click="setMode('road')">快速路隧道</text>
									<text class="ms" :class="{ on: mode === 'metro' }" @click="setMode('metro')">地铁区间</text>
								</view>
								<text class="panel-tag" :class="state.step >= 3 ? 'tag-cyan' : 'tag-green'">{{ sceneTag }}</text>
							</view>
						</view>
						<view class="scene-body">
							<TunnelScene :st="view" :mode="mode" />
						</view>
					</view>
					<LinkFlow :st="view" />
				</view>

				<!-- 右：地图 / 灾害 / 豆包 / 车机 -->
				<view class="col-right">
					<MapBoard :st="view" :mode="mode" :disaster="disasterOn" @open="previewOpen = true" />

					<view class="danger-bar" :class="{ on: disasterOn }" @click="triggerDisaster">
						<text class="db-ico">⚠</text>
						<view class="db-txt">
							<text class="db-t">{{ disasterOn ? '灾情演示进行中 · 点击重演' : '发生灾害 · 一键应急演示' }}</text>
							<text class="db-s">{{ disasterOn ? '火情 → 无人机 → 基站损毁 → 地图推送' : '触发火情剧本，同步推送 3D 现场与研判建议' }}</text>
						</view>
						<text class="db-go">{{ disasterOn ? '重演' : '开始' }}</text>
					</view>

					<AiAdvice :advice="advice" :shown="aiShown" :last-event="lastEvent" />

					<NavAlert :st="view" />
				</view>
			</view>

			<!-- 底部：灾情回溯时间轴 -->
			<view class="timeline">
				<view class="tl-left">
					<text class="tl-btn" :class="{ live: !hist.on }" @click="toggleHist">
						{{ hist.on ? '返回实时' : '灾情回溯' }}
					</text>
					<text class="tl-hint">{{ hist.on ? '拖动查看任一时刻的现场态势' : '实时跟随 · 每 0.5s 一帧' }}</text>
				</view>

				<view class="tl-track">
					<slider class="tl-slider" :min="0" :max="maxIdx" :value="histIdx" :step="1" activeColor="#38bdf8"
						backgroundColor="#17324a" block-size="14" @changing="onScrub" @change="onScrub" />
					<view class="tl-marks">
						<text class="tl-m">T-60s</text>
						<text class="tl-m">T-30s</text>
						<text class="tl-m on">当前</text>
					</view>
				</view>

				<view class="tl-right">
					<view class="tl-stat">
						<text class="ts-k">回溯时间</text>
						<text class="ts-v">{{ histTime }}</text>
					</view>
					<view class="tl-stat">
						<text class="ts-k">帧</text>
						<text class="ts-v">{{ frameLabel }}</text>
					</view>
					<text class="tl-auto" :class="{ on: state.auto }" @click="toggleAuto">
						{{ state.auto ? '自动演示中' : '自动演示' }}
					</text>
				</view>
			</view>

			<view class="footer">
				<text class="ft-brand">烽火燎原队 · 郑州升达经贸管理学院</text>
				<text class="ft-note">演示原型 · 数据由本地剧本引擎生成，不依赖公网与真实硬件</text>
			</view>

			<!-- 现场 3D 重建 · 放大视图 -->
			<view v-if="previewOpen" class="modal-mask" @click="previewOpen = false">
				<view class="modal" @click.stop>
					<view class="modal-hd">
						<view class="mh-l">
							<text class="modal-ic">◈</text>
							<text class="modal-t">现场 3D 重建 · {{ mode === 'metro' ? '地铁区间' : '快速路隧道' }}</text>
							<text class="modal-sub">无人机多视角 + 2.5D 雷达 + 基站传感记录</text>
						</view>
						<text class="modal-x" @click="previewOpen = false">✕</text>
					</view>
					<view class="modal-body">
						<Preview3D :mode="mode" :fire="disasterOn" :flood="disasterOn" />
						<view class="modal-hud">
							<view class="mh-row"><text class="mh-k">点云</text><text class="mh-v">12,480 点</text></view>
							<view class="mh-row"><text class="mh-k">置信度</text><text class="mh-v">92%</text></view>
							<view class="mh-row"><text class="mh-k">对齐误差</text><text class="mh-v">±0.3s</text></view>
							<view class="mh-row"><text class="mh-k">视角</text><text class="mh-v">等轴测 35°</text></view>
						</view>
					</view>
					<view class="modal-ft">
						<text>火源定位：K1+280 第二车道 · 烟气扩散：下风向 40m</text>
						<text>数据来源：AA55 帧 · 3s 周期 · 120s 心跳</text>
					</view>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
	import TunnelScene from '@/components/TunnelScene.vue'
	import MapBoard from '@/components/MapBoard.vue'
	import LinkFlow from '@/components/LinkFlow.vue'
	import AiAdvice from '@/components/AiAdvice.vue'
	import NavAlert from '@/components/NavAlert.vue'
	import Preview3D from '@/components/Preview3D.vue'
	import {
		createState,
		tick,
		setStep,
		STEPS,
		fmtClock
	} from '@/common/sim.js'

	const clamp = (v, a, b) => Math.min(b, Math.max(a, v))
	const DESIGN_W = 1920
	const DESIGN_H = 1080

	export default {
		components: {
			TunnelScene,
			MapBoard,
			LinkFlow,
			AiAdvice,
			NavAlert,
			Preview3D
		},
		data() {
			return {
				state: createState(),
				steps: STEPS,
				scale: 1,
				clock: '--:--:--',
				mode: 'road',
				previewOpen: false,
				hist: {
					on: false,
					idx: 0
				},
				aiShown: '',
				aiTarget: '',
				timer: null
			}
		},
		computed: {
			stageStyle() {
				return 'transform: translate(-50%, -50%) scale(' + this.scale + ')'
			},
			view() {
				if (this.hist.on && this.state.frames.length) {
					const f = this.state.frames[clamp(this.hist.idx, 0, this.state.frames.length - 1)]
					if (f) return f
				}
				return this.state
			},
			disasterOn() {
				return this.state.step >= 2
			},
			onlineCount() {
				const b = this.view.boards
				return (b.base.online ? 1 : 0) + (b.fire.online ? 1 : 0) + (b.flood.online ? 1 : 0)
			},
			hbText() {
				return Math.max(0, Math.round(this.state.hb))
			},
			alertCount() {
				return this.state.alerts.filter((a) => a.level >= 2).length
			},
			sceneTag() {
				const s = this.state.step
				if (s >= 5) return '灾情已发布'
				if (s === 4) return '基站损毁 · 中继中'
				if (s === 3) return '无人机侦察中'
				if (s === 2) return '火情处置中'
				if (s === 1) return '洪涝处置中'
				return '实时孪生'
			},
			maxIdx() {
				return Math.max(0, this.state.frames.length - 1)
			},
			histIdx() {
				return clamp(this.hist.idx, 0, this.maxIdx)
			},
			histTime() {
				const f = this.view
				if (!this.hist.on) return '实时'
				return 'T-' + Math.max(0, Math.round(this.state.t - f.t)) + 's · ' + fmtClock(f.t)
			},
			frameLabel() {
				return (this.hist.on ? this.histIdx + 1 : this.maxIdx + 1) + ' / ' + (this.maxIdx + 1)
			},
			lastEvent() {
				const a = this.state.alerts[0]
				return a ? fmtClock(a.t) + ' ' + a.text : '暂无事件'
			},
			advice() {
				const s = this.state
				const b = s.boards
				const footer = '豆包大模型 · 巴法云 MQTT 接入'
				if (s.step === 0) {
					return {
						active: false,
						tag: '待命',
						input: '',
						summary: '监测中：三块板在线，全部传感器在阈值内。点击右侧「发生灾害」可触发一次完整应急演示，我会同步给出研判结论与处置建议。',
						items: [],
						footer: footer
					}
				}
				if (s.step === 1) {
					const d = Math.max(0, Math.round((60 - b.flood.dist) * 1.2))
					return {
						active: true,
						tag: '洪涝 · 三级',
						input: '洪涝板上报：超声波 ' + b.flood.dist.toFixed(0) + 'cm，水浸 ' + (b.flood.wet ? '已触发' : '干燥') +
							'，PIR ' + (b.flood.pir ? '检测到人员' : '无人') + '，舵机 ' + b.flood.servo + '°。',
						summary: '判定为隧道积水漫流，水位已越过三级阈值（积水约 ' + d + 'cm）。建议全力排水、封闭入口，并引导后方车辆改道，避免盲目跟进积水段。',
						items: [{
								k: '排水',
								v: '挡板 ' + b.flood.servo + '°，全速排水'
							},
							{
								k: '警示',
								v: b.flood.buzzer ? '蜂鸣器周期鸣响 + 语音疏散' : '暂未触发'
							},
							{
								k: '交通',
								v: '入口封闭，辅路绕行'
							}
						],
						footer: footer
					}
				}
				const drop = Math.round((b.base.pressRef - b.base.press) * 100)
				const items = [{
					k: '排烟',
					v: b.fire.fanDir + ' ' + b.fire.fanPwm + '% · ' + b.fire.lcd
				}]
				if (s.step >= 3) {
					items.push({
						k: '侦察',
						v: '无人机 ' + s.drone.alt.toFixed(1) + 'm 定高 · ' + s.drone.link
					})
				}
				items.push({
					k: '疏散',
					v: '语音播报 + 应急灯带，向上风向出口撤离'
				})
				items.push({
					k: '交通',
					v: s.step >= 5 ? '已推送导航端，建议辅路绕行' : '准备推送导航端'
				})
				if (s.step >= 4) {
					items.push({
						k: '链路',
						v: '基站损毁，LoRa 经无人机中继，数据不中断'
					})
				}
				return {
					active: true,
					tag: '火情 · 三级',
					input: '火情板上报：MQ2 ' + b.fire.smoke.toFixed(0) + '%、MQ7 ' + b.fire.co.toFixed(0) + '%、MQ135 ' +
						b.fire.air.toFixed(0) + '%、火焰 ' + b.fire.flame.toFixed(0) + '%，气压骤降 ' + drop + 'Pa。',
					summary: '多源数据交叉确认为车辆燃烧且已越过三级排烟阈值，烟气正在向下风向扩散。建议满负荷正转排烟、立即封闭车道，并组织下风向人员经上风向出口撤离。',
					items: items,
					footer: footer
				}
			},
			cards() {
				const b = this.view.boards
				const base = b.base
				const fire = b.fire
				const flood = b.flood
				const drop = (base.pressRef - base.press) * 100
				const tilt = Math.sqrt(base.pitch * base.pitch + base.roll * base.roll)
				return [{
						key: 'base',
						icon: '⌂',
						name: '地面基站板',
						sub: 'LoRa 汇聚 · MS5611 · MPU6050 · SD',
						online: base.online,
						tone: base.online ? 'ok' : 'bad',
						badge: base.online ? '在线' : '损毁',
						rows: [
							{ k: 'LoRa RSSI', v: base.rssi.toFixed(0), u: 'dBm', pct: clamp(((-base.rssi) - 60) / 50 * 100, 0, 100), tone: base.rssi < -115 ? 'warn' : 'ok' },
							{ k: '链路丢包', v: base.loss.toFixed(1), u: '%', pct: clamp(base.loss * 6, 0, 100), tone: base.loss > 5 ? 'warn' : 'ok' },
							{ k: 'MS5611 气压', v: base.press.toFixed(1), u: 'hPa', pct: clamp((base.press - 990) / 40 * 100, 0, 100), tone: 'info' },
							{ k: '气压骤差', v: drop.toFixed(0), u: 'Pa', pct: clamp(drop / 700 * 100, 0, 100), tick: 71, tone: drop > 500 ? 'bad' : 'info' },
							{ k: 'MPU6050 倾角', v: tilt.toFixed(1), u: '°', pct: clamp(tilt / 20 * 100, 0, 100), tick: 40, tone: tilt > 8 ? 'bad' : 'ok' },
							{ k: 'SD 卡 / 上行', v: base.sd, u: '', pct: null, tone: base.sd === '写入中' ? 'warn' : 'info' },
							{ k: 'WiFi', v: base.wifi, u: '', pct: null, tone: base.online ? 'ok' : 'bad' }
						],
						foot: ['帧头 AA 55', '心跳 ' + this.hbText + 's', 'SD ' + Math.round(base.sdBytes) + 'KB']
					},
					{
						key: 'fire',
						icon: '⚑',
						name: '火情监测板',
						sub: 'MQ2 / MQ7 / MQ135 · 火焰 · 风机',
						online: fire.online,
						tone: fire.level >= 3 ? 'bad' : (fire.level > 0 ? 'warn' : 'ok'),
						badge: ['正常', '一级', '二级', '三级'][fire.level],
						rows: [
							{ k: 'MQ2 烟雾', v: fire.smoke.toFixed(0), u: '%', pct: clamp(fire.smoke, 0, 100), tick: 65, tone: fire.smoke > 65 ? 'bad' : 'ok' },
							{ k: 'MQ7 一氧化碳', v: fire.co.toFixed(0), u: '%', pct: clamp(fire.co, 0, 100), tick: 55, tone: fire.co > 55 ? 'bad' : 'ok' },
							{ k: 'MQ135 空气质量', v: fire.air.toFixed(0), u: '%', pct: clamp(fire.air, 0, 100), tick: 40, tone: fire.air > 40 ? 'warn' : 'ok' },
							{ k: '火焰强度', v: fire.flame.toFixed(0), u: '%', pct: clamp(fire.flame * 2, 0, 100), tick: 40, tone: fire.flame > 20 ? 'bad' : 'ok' },
							{ k: '环境温度', v: fire.temp.toFixed(1), u: '℃', pct: clamp((fire.temp - 20) / 45 * 100, 0, 100), tone: fire.temp > 45 ? 'warn' : 'info' },
							{ k: '风机', v: fire.fanDir + ' ' + fire.fanPwm, u: '%', pct: fire.fanPwm, tone: fire.fanPwm > 0 ? 'warn' : 'info' },
							{ k: 'LCD', v: fire.lcd, u: '', pct: null, tone: fire.level > 0 ? 'warn' : 'ok' }
						],
						foot: ['ESP-NOW 上行', '去极值均值滤波', fire.pre ? '气压骤降·超前预警' : '阈值 65/55/40']
					},
					{
						key: 'flood',
						icon: '≈',
						name: '洪涝监测板',
						sub: '超声波 · 水位 · PIR · SG90 · LoRa',
						online: flood.online,
						tone: flood.level >= 3 ? 'bad' : (flood.level > 0 ? 'warn' : 'ok'),
						badge: ['安全', '一级', '二级', '三级'][flood.level],
						rows: [
							{ k: '超声波测距', v: flood.dist.toFixed(1), u: 'cm', pct: clamp((80 - flood.dist) / 70 * 100, 0, 100), tick: 71, tone: flood.level > 0 ? 'warn' : 'ok' },
							{ k: '接触式水位', v: flood.wet ? '已触发' : '干燥', u: '', pct: null, tone: flood.wet ? 'bad' : 'ok' },
							{ k: 'PIR 人体', v: flood.pir ? '有人' : '无人', u: '', pct: null, tone: flood.pir ? 'warn' : 'info' },
							{ k: 'SG90 舵机', v: flood.servo, u: '°', pct: flood.servo / 180 * 100, tone: flood.servo > 0 ? 'warn' : 'ok' },
							{ k: '蜂鸣器', v: flood.buzzer ? '鸣响' : '静音', u: '', pct: null, tone: flood.buzzer ? 'bad' : 'ok' },
							{ k: 'BMP280 气压', v: flood.press.toFixed(1), u: 'hPa', pct: clamp((flood.press - 990) / 40 * 100, 0, 100), tone: 'info' },
							{ k: 'LCD', v: flood.lcd, u: '', pct: null, tone: flood.level > 0 ? 'warn' : 'ok' }
						],
						foot: ['LoRa 上行', '冒泡去极值', '三级 60/40/20cm']
					}
				]
			}
		},
		mounted() {
			this.fit()
			if (typeof window !== 'undefined' && window.addEventListener) {
				window.addEventListener('resize', this.fit)
			}
			this.tickClock()
			this.timer = setInterval(() => {
				if (!this.hist.on) tick(this.state, 0.25)
				if (this.hist.on) this.hist.idx = clamp(this.hist.idx, 0, this.maxIdx)
				this.tickClock()
				if (this.aiTarget !== this.advice.summary) {
					this.aiTarget = this.advice.summary
					this.aiShown = ''
				}
				if (this.aiShown.length < this.aiTarget.length) {
					this.aiShown = this.aiTarget.slice(0, this.aiShown.length + 5)
				}
			}, 250)
		},
		beforeDestroy() {
			if (this.timer) clearInterval(this.timer)
			if (typeof window !== 'undefined' && window.removeEventListener) {
				window.removeEventListener('resize', this.fit)
			}
		},
		methods: {
			fit() {
				if (typeof window === 'undefined') return
				const w = window.innerWidth || DESIGN_W
				const h = window.innerHeight || DESIGN_H
				this.scale = Math.min(w / DESIGN_W, h / DESIGN_H)
			},
			tickClock() {
				const d = new Date()
				const p = (v) => (v < 10 ? '0' + v : '' + v)
				this.clock = p(d.getHours()) + ':' + p(d.getMinutes()) + ':' + p(d.getSeconds())
			},
			go(i) {
				setStep(this.state, i)
				this.hist.on = false
				this.hist.idx = 0
			},
			setMode(m) {
				this.mode = m
			},
			triggerDisaster() {
				setStep(this.state, 2)
				this.state.auto = true
				this.hist.on = false
				this.hist.idx = 0
			},
			toggleAuto() {
				this.state.auto = !this.state.auto
			},
			toggleHist() {
				this.hist.on = !this.hist.on
				if (this.hist.on) this.hist.idx = this.maxIdx
			},
			onScrub(e) {
				this.hist.on = true
				this.hist.idx = Number(e.detail.value)
			}
		}
	}
</script>

<style>
	.screen {
		position: fixed;
		left: 0;
		top: 0;
		right: 0;
		bottom: 0;
		overflow: hidden;
		background: #04080e;
	}

	.stage {
		position: absolute;
		left: 50%;
		top: 50%;
		width: 1920px;
		height: 1080px;
		transform-origin: center center;
		padding: 14px 18px;
		box-sizing: border-box;
		display: flex;
		flex-direction: column;
		gap: 12px;
		background:
			radial-gradient(1200px 600px at 18% -10%, rgba(24, 90, 130, 0.28), transparent 60%),
			radial-gradient(900px 500px at 88% 8%, rgba(120, 60, 30, 0.16), transparent 60%),
			#04080e;
	}

	/* ---------------- 顶栏 ---------------- */
	.topbar {
		flex: 0 0 66px;
		display: flex;
		align-items: center;
		gap: 20px;
		padding: 0 14px;
		border: 1px solid rgba(56, 189, 248, 0.14);
		border-radius: 6px;
		background: linear-gradient(180deg, rgba(10, 24, 38, 0.9), rgba(6, 13, 21, 0.85));
	}

	.brand {
		display: flex;
		align-items: center;
		gap: 10px;
	}

	.logo {
		width: 40px;
		height: 40px;
	}

	.brand-txt {
		display: flex;
		flex-direction: column;
	}

	.brand-line {
		display: flex;
		align-items: baseline;
		gap: 8px;
	}

	.brand-name {
		font-size: 22px;
		font-weight: 700;
		color: #e8f5ff;
		letter-spacing: 3px;
	}

	.brand-en {
		font-size: 9px;
		color: #3f7f9e;
		letter-spacing: 2px;
	}

	.brand-sub {
		font-size: 10.5px;
		color: #5d7d94;
		margin-top: 2px;
		letter-spacing: 0.5px;
	}

	.steps {
		display: flex;
		align-items: center;
		gap: 6px;
		margin-left: auto;
	}

	.step {
		display: flex;
		align-items: center;
		gap: 6px;
		padding: 6px 12px;
		border-radius: 4px;
		border: 1px solid rgba(56, 189, 248, 0.16);
		background: rgba(10, 26, 40, 0.5);
		cursor: pointer;
		transition: all 0.2s;
	}

	.step.done {
		border-color: rgba(52, 211, 153, 0.3);
		background: rgba(11, 40, 32, 0.55);
	}

	.step.on {
		border-color: rgba(56, 189, 248, 0.85);
		background: rgba(14, 60, 90, 0.85);
		box-shadow: 0 0 14px rgba(56, 189, 248, 0.25);
	}

	.step-i {
		font-size: 9px;
		color: #3d6d88;
		font-family: Consolas, monospace;
	}

	.step.on .step-i {
		color: #7fe3ff;
	}

	.step-n {
		font-size: 12px;
		color: #7f9cb1;
	}

	.step.on .step-n {
		color: #dff4ff;
	}

	.topright {
		display: flex;
		align-items: center;
		gap: 10px;
	}

	.chip {
		display: flex;
		align-items: center;
		gap: 5px;
		padding: 4px 9px;
		border-radius: 3px;
		border: 1px solid rgba(56, 189, 248, 0.14);
		background: rgba(9, 22, 34, 0.7);
	}

	.chip.danger {
		border-color: rgba(255, 77, 79, 0.45);
		background: rgba(60, 16, 18, 0.6);
	}

	.c-k {
		font-size: 9.5px;
		color: #4f7288;
	}

	.c-v {
		font-size: 12px;
		color: #cfe6f5;
		font-family: Consolas, monospace;
	}

	.chip.danger .c-v {
		color: #ff8a8c;
	}

	.clock {
		font-size: 15px;
		color: #9dd8f2;
		font-family: Consolas, monospace;
		letter-spacing: 1px;
	}

	/* ---------------- 主区 ---------------- */
	.main {
		flex: 1;
		min-height: 0;
		display: flex;
		gap: 12px;
	}

	.col-left {
		flex: 0 0 372px;
		display: flex;
		flex-direction: column;
		gap: 12px;
		min-height: 0;
	}

	.col-mid {
		flex: 1;
		min-width: 0;
		display: flex;
		flex-direction: column;
		gap: 10px;
	}

	.col-right {
		flex: 0 0 422px;
		display: flex;
		flex-direction: column;
		gap: 12px;
		min-height: 0;
	}

	.scene-card {
		flex: 1;
		min-height: 0;
		display: flex;
		flex-direction: column;
		border: 1px solid rgba(56, 189, 248, 0.14);
		border-radius: 6px;
		background: rgba(7, 15, 24, 0.85);
		overflow: hidden;
	}

	.scene-body {
		flex: 1;
		min-height: 0;
		padding: 4px 6px 6px;
	}

	.hd-right {
		margin-left: auto;
		display: flex;
		align-items: center;
		gap: 10px;
	}

	.hd-right .panel-tag {
		margin-left: 0;
	}

	.mode-switch {
		display: flex;
		border: 1px solid rgba(56, 189, 248, 0.22);
		border-radius: 4px;
		overflow: hidden;
	}

	.ms {
		font-size: 10.5px;
		padding: 3px 10px;
		color: #5f8098;
		cursor: pointer;
		transition: all 0.2s;
	}

	.ms.on {
		background: rgba(56, 189, 248, 0.9);
		color: #04121a;
		font-weight: 600;
	}

	/* ---------------- 板卡 ---------------- */
	.board {
		flex: 1;
		min-height: 0;
		display: flex;
		flex-direction: column;
		padding: 8px 12px 6px;
		border-radius: 6px;
		border: 1px solid rgba(56, 189, 248, 0.13);
		border-left: 2px solid #2f6f8f;
		background: linear-gradient(180deg, rgba(10, 23, 36, 0.92), rgba(6, 13, 21, 0.9));
	}

	.board.b-warn {
		border-left-color: #fbbf24;
		box-shadow: inset 0 0 30px rgba(251, 191, 36, 0.05);
	}

	.board.b-bad {
		border-left-color: #ff4d4f;
		box-shadow: inset 0 0 30px rgba(255, 77, 79, 0.08);
	}

	.b-hd {
		display: flex;
		align-items: center;
		gap: 9px;
		padding-bottom: 6px;
		border-bottom: 1px dashed rgba(56, 189, 248, 0.12);
	}

	.b-ico {
		font-size: 16px;
		color: #6fb9dd;
	}

	.b-tt {
		display: flex;
		flex-direction: column;
	}

	.b-name {
		font-size: 13.5px;
		color: #d6ecfa;
		font-weight: 600;
		letter-spacing: 0.6px;
	}

	.b-sub {
		font-size: 9.5px;
		color: #4d6c82;
		margin-top: 1px;
	}

	.b-pill {
		margin-left: auto;
		font-size: 10px;
		padding: 2px 8px;
		border-radius: 9px;
		border: 1px solid rgba(52, 211, 153, 0.35);
		color: #6ee7b7;
	}

	.b-pill.p-warn {
		border-color: rgba(251, 191, 36, 0.4);
		color: #fcd34d;
	}

	.b-pill.p-bad {
		border-color: rgba(255, 77, 79, 0.5);
		color: #ff8a8c;
	}

	.b-rows {
		flex: 1;
		display: flex;
		flex-direction: column;
		justify-content: space-around;
		padding: 2px 0;
	}

	.row {
		display: flex;
		align-items: center;
		gap: 8px;
		height: 22px;
	}

	.r-k {
		width: 96px;
		font-size: 10.5px;
		color: #6b8aa1;
		letter-spacing: 0.3px;
	}

	.r-mid {
		flex: 1;
		display: flex;
		align-items: center;
		gap: 8px;
		min-width: 0;
	}

	.r-bar {
		position: relative;
		flex: 1;
		height: 5px;
		border-radius: 3px;
		background: rgba(38, 66, 88, 0.55);
	}

	.r-fill {
		height: 100%;
		border-radius: 3px;
		background: #2f8fb5;
		transition: width 0.25s linear;
	}

	.r-fill.f-ok {
		background: linear-gradient(90deg, #1d6b52, #34d399);
	}

	.r-fill.f-warn {
		background: linear-gradient(90deg, #7a5a12, #fbbf24);
	}

	.r-fill.f-bad {
		background: linear-gradient(90deg, #7a1f21, #ff4d4f);
	}

	.r-fill.f-info {
		background: linear-gradient(90deg, #1c4a63, #38bdf8);
	}

	.r-tick {
		position: absolute;
		top: -3px;
		width: 1px;
		height: 11px;
		background: rgba(255, 255, 255, 0.35);
	}

	.r-v {
		width: 82px;
		text-align: right;
		font-size: 12px;
		color: #c4dcec;
		font-family: Consolas, monospace;
		flex: 0 0 auto;
	}

	.r-v.v-ok {
		color: #8fe9c4;
	}

	.r-v.v-warn {
		color: #fcd34d;
	}

	.r-v.v-bad {
		color: #ff8a8c;
	}

	.r-u {
		font-size: 9px;
		color: #527087;
		margin-left: 2px;
	}

	.b-ft {
		display: flex;
		gap: 12px;
		padding-top: 5px;
		border-top: 1px dashed rgba(56, 189, 248, 0.1);
	}

	.ft-i {
		font-size: 9px;
		color: #3f5c72;
		font-family: Consolas, monospace;
	}

	/* ---------------- 灾害按钮 ---------------- */
	.danger-bar {
		flex: 0 0 auto;
		height: 52px;
		display: flex;
		align-items: center;
		gap: 10px;
		padding: 0 12px;
		border-radius: 6px;
		border: 1px solid rgba(255, 77, 79, 0.45);
		background: linear-gradient(100deg, rgba(120, 20, 24, 0.55), rgba(60, 16, 20, 0.35));
		cursor: pointer;
		transition: all 0.2s;
	}

	.danger-bar:hover {
		border-color: rgba(255, 77, 79, 0.8);
		background: linear-gradient(100deg, rgba(150, 26, 30, 0.65), rgba(70, 18, 22, 0.45));
	}

	.danger-bar.on {
		border-color: rgba(255, 138, 61, 0.7);
		animation: dbPulse 2.2s ease-in-out infinite;
	}

	@keyframes dbPulse {
		0%, 100% { box-shadow: 0 0 0 rgba(255, 77, 79, 0); }
		50% { box-shadow: 0 0 18px rgba(255, 77, 79, 0.3); }
	}

	.db-ico {
		font-size: 20px;
		color: #ff8a8c;
	}

	.db-txt {
		flex: 1;
		display: flex;
		flex-direction: column;
	}

	.db-t {
		font-size: 12.5px;
		color: #ffd9c9;
		font-weight: 600;
		letter-spacing: 0.5px;
	}

	.db-s {
		font-size: 9.5px;
		color: #a8736c;
		margin-top: 1px;
	}

	.db-go {
		font-size: 10.5px;
		color: #fff;
		background: rgba(255, 77, 79, 0.9);
		padding: 4px 12px;
		border-radius: 3px;
		font-weight: 600;
	}

	/* ---------------- 面板通用 ---------------- */
	.panel {
		display: flex;
		flex-direction: column;
		border: 1px solid rgba(56, 189, 248, 0.14);
		border-radius: 6px;
		background: linear-gradient(180deg, rgba(10, 23, 36, 0.9), rgba(6, 13, 21, 0.9));
		overflow: hidden;
	}

	.panel-hd {
		flex: 0 0 auto;
		display: flex;
		align-items: center;
		gap: 10px;
		padding: 7px 12px;
		border-bottom: 1px solid rgba(56, 189, 248, 0.1);
	}

	.panel-title {
		font-size: 12px;
		color: #b9d7ea;
		letter-spacing: 1.2px;
	}

	.panel-tag {
		margin-left: auto;
		font-size: 9.5px;
		padding: 2px 8px;
		border-radius: 9px;
		border: 1px solid rgba(56, 189, 248, 0.25);
		color: #7fc7ea;
	}

	.panel-tag.tag-green {
		color: #6ee7b7;
		border-color: rgba(52, 211, 153, 0.35);
	}

	.panel-tag.tag-red {
		color: #ff8a8c;
		border-color: rgba(255, 77, 79, 0.45);
		background: rgba(255, 77, 79, 0.08);
	}

	.panel-tag.tag-cyan {
		color: #7fe3ff;
	}

	.panel-tag.tag-dim {
		color: #55718a;
		border-color: rgba(85, 113, 138, 0.3);
	}

	/* ---------------- 时间轴 ---------------- */
	.timeline {
		flex: 0 0 62px;
		display: flex;
		align-items: center;
		gap: 16px;
		padding: 0 14px;
		border: 1px solid rgba(56, 189, 248, 0.14);
		border-radius: 6px;
		background: linear-gradient(180deg, rgba(10, 24, 38, 0.9), rgba(6, 13, 21, 0.85));
	}

	.tl-left {
		display: flex;
		flex-direction: column;
		gap: 3px;
		width: 200px;
	}

	.tl-btn {
		font-size: 12px;
		color: #7fc7ea;
		padding: 3px 10px;
		border: 1px solid rgba(56, 189, 248, 0.4);
		border-radius: 3px;
		text-align: center;
		cursor: pointer;
	}

	.tl-btn.live {
		color: #6ee7b7;
		border-color: rgba(52, 211, 153, 0.35);
	}

	.tl-hint {
		font-size: 9px;
		color: #3f5c72;
	}

	.tl-track {
		flex: 1;
		min-width: 0;
		display: flex;
		flex-direction: column;
	}

	.tl-slider {
		width: 100%;
		margin: 0;
	}

	.tl-marks {
		display: flex;
		justify-content: space-between;
		padding: 0 4px;
	}

	.tl-m {
		font-size: 9px;
		color: #3f5c72;
		font-family: Consolas, monospace;
	}

	.tl-m.on {
		color: #7fe3ff;
	}

	.tl-right {
		display: flex;
		align-items: center;
		gap: 18px;
	}

	.tl-stat {
		display: flex;
		flex-direction: column;
		gap: 2px;
	}

	.ts-k {
		font-size: 9px;
		color: #3f5c72;
	}

	.ts-v {
		font-size: 12px;
		color: #cfe6f5;
		font-family: Consolas, monospace;
	}

	.tl-auto {
		font-size: 11px;
		color: #7f9cb1;
		padding: 4px 10px;
		border: 1px dashed rgba(56, 189, 248, 0.28);
		border-radius: 3px;
		cursor: pointer;
	}

	.tl-auto.on {
		color: #08131d;
		background: #38bdf8;
		border-style: solid;
		border-color: #38bdf8;
		font-weight: 600;
	}

	/* ---------------- 页脚 ---------------- */
	.footer {
		flex: 0 0 16px;
		display: flex;
		align-items: center;
		gap: 20px;
		padding: 0 4px;
	}

	.ft-brand {
		font-size: 9.5px;
		color: #35566d;
		letter-spacing: 0.5px;
	}

	.ft-note {
		font-size: 9px;
		color: #2b4356;
		margin-left: auto;
	}

	/* ---------------- 3D 放大视图 ---------------- */
	.modal-mask {
		position: absolute;
		left: 0;
		top: 0;
		right: 0;
		bottom: 0;
		background: rgba(2, 6, 12, 0.78);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 50;
	}

	.modal {
		width: 820px;
		border-radius: 8px;
		border: 1px solid rgba(255, 138, 61, 0.45);
		background: linear-gradient(180deg, rgba(10, 18, 28, 0.98), rgba(5, 10, 16, 0.98));
		box-shadow: 0 30px 80px rgba(0, 0, 0, 0.8), 0 0 40px rgba(255, 90, 40, 0.12);
		overflow: hidden;
	}

	.modal-hd {
		display: flex;
		align-items: center;
		padding: 10px 14px;
		border-bottom: 1px solid rgba(255, 138, 61, 0.2);
		background: rgba(255, 138, 61, 0.06);
	}

	.mh-l {
		display: flex;
		align-items: baseline;
		gap: 10px;
		flex: 1;
	}

	.modal-ic {
		font-size: 14px;
		color: #ff8a3d;
	}

	.modal-t {
		font-size: 14px;
		color: #ffd9c9;
		letter-spacing: 1.2px;
	}

	.modal-sub {
		font-size: 10px;
		color: #7a6558;
	}

	.modal-x {
		font-size: 14px;
		color: #8b7568;
		padding: 2px 8px;
		cursor: pointer;
	}

	.modal-body {
		position: relative;
		height: 480px;
		background: #061019;
	}

	.modal-hud {
		position: absolute;
		right: 12px;
		top: 12px;
		display: flex;
		flex-direction: column;
		gap: 4px;
		padding: 8px 10px;
		border-radius: 4px;
		background: rgba(3, 8, 14, 0.7);
		border: 1px solid rgba(56, 189, 248, 0.2);
	}

	.mh-row {
		display: flex;
		justify-content: space-between;
		gap: 14px;
		line-height: 15px;
	}

	.mh-k {
		font-size: 9.5px;
		color: #5f7f96;
	}

	.mh-v {
		font-size: 10px;
		color: #b9e4ff;
		font-family: Consolas, monospace;
	}

	.modal-ft {
		display: flex;
		justify-content: space-between;
		gap: 10px;
		padding: 8px 14px;
		border-top: 1px solid rgba(56, 189, 248, 0.1);
		font-size: 9.5px;
		color: #4f6a80;
		font-family: Consolas, monospace;
	}
</style>

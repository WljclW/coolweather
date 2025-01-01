@Override
    public List<JSONObject> filterActionValueWithStdRuleId(List<Device> devices, int stdRuleId) {
        String tag = "filterA";
        List<JSONObject> result = new ArrayList<>();
        ConcurrentHashMap<String, CopyOnWriteArrayList<Integer>> actionsDid = new ConcurrentHashMap<>();//SpecSceneConfigRepository.INSTANCE.getActionsDid();
        ConcurrentHashMap<String, CopyOnWriteArrayList<String>> stdActionsDid = new ConcurrentHashMap<>();//SpecSceneConfigRepository.INSTANCE.getActionStdDid();
        for(Device d:devices){
            CopyOnWriteArrayList<Integer> tmpActions = SpecSceneConfigRepository.INSTANCE.getActionsDid().get(d.did);
            if(tmpActions!=null){
                actionsDid.put(d.did, tmpActions);
            }
            CopyOnWriteArrayList<String> tmpStdActions = SpecSceneConfigRepository.INSTANCE.getActionStdDid().get(d.did);
            if (tmpStdActions != null) {
                stdActionsDid.put(d.did, tmpStdActions);
            }

        }
        ConcurrentHashMap<Integer, ATplItem> actionsOnline = SpecSceneConfigRepository.INSTANCE.getActionsOnline();
        ConcurrentHashMap<String, ATplItem> stdActionsOnline = SpecSceneConfigRepository.INSTANCE.getActionsStd();
        JSONObject oneItem;
        for (Map.Entry<String, CopyOnWriteArrayList<Integer>> entry:actionsDid.entrySet()) {
            String did = entry.getKey();
            Device d = SpecSceneHelper.Companion.findDeviceById(did);
            if(d == null){
                MiJiaLog.onlyLogcat(tag,"empty device :: "+did);
                continue;
            }
            List<Integer> saIds = actionsDid.get(did);
            if(saIds == null){
                continue;
            }

            for (Integer saId : saIds) {
                ATplItem aTplItem = actionsOnline.get(saId);
                if(aTplItem == null) {
                    continue;
                }
                MiJiaLog.onlyLogcat(tag, "actionOnline:  "+d.name+"   "+aTplItem.getStdRuleId()+"   :: :: " + stdRuleId);
                if(aTplItem.getStdRuleId()!=stdRuleId){
                    continue;
                }
                oneItem = buildA(aTplItem,d);
                if (oneItem != null) {
                    result.add(oneItem);
                }
            }
        }

        for (Map.Entry<String, CopyOnWriteArrayList<String>> entry:stdActionsDid.entrySet()) {
            String did = entry.getKey();
            Device d = SpecSceneHelper.Companion.findDeviceById(did);
            if(d == null){
                MiJiaLog.onlyLogcat(tag,"empty device :: "+did);
                continue;
            }
            List<String> saIds = stdActionsDid.get(d.did);
            if(saIds == null){
                continue;
            }

            for (String saId : saIds) {
                ATplItem aTplItem = stdActionsOnline.get(saId);
                if(aTplItem == null) {
                    continue;
                }
                MiJiaLog.onlyLogcat(tag, "actionStd:  "+d.name+"  "+aTplItem.getStdRuleId()+"   :: :: " + stdRuleId);
                if(aTplItem.getStdRuleId()!=stdRuleId){
                    continue;
                }
                oneItem = buildA(aTplItem,d);
                if (oneItem != null) {
                    result.add(oneItem);
                }
            }
        }


        MiJiaLog.onlyLogcat("jmd236sss","filterActionValueWithStdRuleId: "+result.size());
        return result;
    }

==============================================================================================
package com.xiaomi.smarthome.specscene.util

import android.app.Activity
import android.content.Context
import android.content.DialogInterface
import android.content.Intent
import android.graphics.drawable.BitmapDrawable
import android.net.Uri
import android.os.Build
import android.os.Bundle
import android.provider.Settings
import android.text.TextUtils
import android.view.Gravity
import androidx.activity.result.ActivityResultLauncher
import androidx.fragment.app.FragmentActivity
import androidx.lifecycle.LifecycleOwner
import androidx.localbroadcastmanager.content.LocalBroadcastManager
import com.facebook.drawee.backends.pipeline.Fresco
import com.facebook.drawee.controller.BaseControllerListener
import com.facebook.drawee.drawable.ScalingUtils
import com.facebook.drawee.generic.GenericDraweeHierarchyBuilder
import com.facebook.drawee.interfaces.DraweeController
import com.facebook.drawee.view.SimpleDraweeView
import com.facebook.imagepipeline.common.ResizeOptions
import com.facebook.imagepipeline.request.ImageRequestBuilder
import com.miui.autotask.bean.SafeCenterHelper
import com.miui.autotask.bean.Task
import com.xiaomi.hmind2.entity.HmAuthEntity
import com.xiaomi.smarthome.AppStateNotifier
import com.xiaomi.smarthome.abtest.ABConfig
import com.xiaomi.smarthome.abtest.ABTesting
import com.xiaomi.smarthome.api.geofence.IMIUIGeoFenceManagerApi
import com.xiaomi.smarthome.application.CommonApplication
import com.xiaomi.smarthome.application.ServiceApplication
import com.xiaomi.smarthome.autotask.bean.Condition
import com.xiaomi.smarthome.camera.api.CameraRouterFactory
import com.xiaomi.smarthome.card.CardRouterFactory
import com.xiaomi.smarthome.component.MiJiaRouter
import com.xiaomi.smarthome.core.entity.plugin.PluginDownloadTask
import com.xiaomi.smarthome.device.BleDevice
import com.xiaomi.smarthome.device.Device
import com.xiaomi.smarthome.device.DeviceFilter
import com.xiaomi.smarthome.device.DeviceRouterFactory
import com.xiaomi.smarthome.device.SmartHomeDeviceHelper
import com.xiaomi.smarthome.device.SmartHomeDeviceManager
import com.xiaomi.smarthome.device.api.IXmPluginMessageReceiver
import com.xiaomi.smarthome.device.api.spec.SpecUtils
import com.xiaomi.smarthome.device.api.spec.instance.Spec
import com.xiaomi.smarthome.device.api.spec.instance.SpecDevice
import com.xiaomi.smarthome.frame.AsyncCallback
import com.xiaomi.smarthome.frame.AsyncHandle
import com.xiaomi.smarthome.frame.Error
import com.xiaomi.smarthome.frame.cancel.CancelableRef
import com.xiaomi.smarthome.frame.core.CoreApi
import com.xiaomi.smarthome.frame.plugin.PluginApi
import com.xiaomi.smarthome.frame.plugin.SendMessageCallback
import com.xiaomi.smarthome.frame.server_compact.ServerBean
import com.xiaomi.smarthome.frame.server_compact.ServerCompact
import com.xiaomi.smarthome.framework.page.BaseActivity
import com.xiaomi.smarthome.framework.page.CommonActivity
import com.xiaomi.smarthome.framework.page.DevelopSharePreManager
import com.xiaomi.smarthome.framework.page.rndebug.RnDebugConstant
import com.xiaomi.smarthome.framework.page.rndebug.RnDebugFileUtil
import com.xiaomi.smarthome.hm_external.IPeopleCarHomeDialog
import com.xiaomi.smarthome.homeroom.CarManager
import com.xiaomi.smarthome.homeroom.CarManager.getCarRoomByDid
import com.xiaomi.smarthome.homeroom.CarManager.getCarRoomByHomeId
import com.xiaomi.smarthome.homeroom.CarManager.whichCarRoomTempDevice
import com.xiaomi.smarthome.homeroom.HomeManager
import com.xiaomi.smarthome.homeroom.WallPaperConfig
import com.xiaomi.smarthome.homeroom.WallPaperUtils
import com.xiaomi.smarthome.homeroom.model.CarRoomData
import com.xiaomi.smarthome.homeroom.model.Home
import com.xiaomi.smarthome.homeroom.model.IHomeManagerApi
import com.xiaomi.smarthome.library.common.dialog.MLAlertDialog
import com.xiaomi.smarthome.library.common.dialog.XQProgressHorizontalDialog
import com.xiaomi.smarthome.library.common.dialogx.builder.scene.MJOptionClickDialog
import com.xiaomi.smarthome.library.common.dialogx.builder.scene.OptionClickViewContent
import com.xiaomi.smarthome.library.common.dialogx.builder.text.MJMessageDialog
import com.xiaomi.smarthome.library.common.util.CommonUtil
import com.xiaomi.smarthome.library.common.util.SharePrefsManager
import com.xiaomi.smarthome.library.common.util.ToastUtil
import com.xiaomi.smarthome.library.crypto.MD5Util
import com.xiaomi.smarthome.library.deviceId.IdentifierManager
import com.xiaomi.smarthome.library.log.LogType
import com.xiaomi.smarthome.library.log.MiJiaLog
import com.xiaomi.smarthome.mainpage.external.IMainPageApiProvider
import com.xiaomi.smarthome.newui.utils.DeviceUtils
import com.xiaomi.smarthome.plugin.update.PluginError
import com.xiaomi.smarthome.scene.BuildConfig
import com.xiaomi.smarthome.scene.ConstantType
import com.xiaomi.smarthome.scene.ConstantType.KEY_CUSTOM_HIDE_OR_GRAY
import com.xiaomi.smarthome.scene.R
import com.xiaomi.smarthome.scene.activity.SmartHomeSceneUtility
import com.xiaomi.smarthome.scene.activity.SmarthomeCreateAutoSceneActivity
import com.xiaomi.smarthome.scene.api.BaseSceneApi
import com.xiaomi.smarthome.scene.api.IScenceListener
import com.xiaomi.smarthome.scene.api.RemoteApi
import com.xiaomi.smarthome.scene.api.RemoteSceneInnerApi
import com.xiaomi.smarthome.scene.api.SceneApi.SmartHomeScene
import com.xiaomi.smarthome.scene.bean.CommonUsedScene
import com.xiaomi.smarthome.scene.bean.RecommendTpl
import com.xiaomi.smarthome.scene.bean.RecommendTplCategory
import com.xiaomi.smarthome.scene.bean.SortSceneData
import com.xiaomi.smarthome.scene.bean.WidgetScene
import com.xiaomi.smarthome.scene.bridge.SceneRouterFactory
import com.xiaomi.smarthome.scene.geofence.manager.MIUIGeoFenceManager
import com.xiaomi.smarthome.scene.manager.PluginRecommendSceneManager
import com.xiaomi.smarthome.scene.manager.RecommendSceneManager
import com.xiaomi.smarthome.scene.manager.SceneRuleCaculator
import com.xiaomi.smarthome.scene.model.CorntabUtils
import com.xiaomi.smarthome.scene.model.CorntabUtils.CorntabParam
import com.xiaomi.smarthome.scene.model.scene.SimpleScene
import com.xiaomi.smarthome.scene.model.scene.SpecScene
import com.xiaomi.smarthome.scene.model.scene.SpecScene.Action.QuickActionPayload
import com.xiaomi.smarthome.scene.model.scene.SpecScene.Action.RpcPayload
import com.xiaomi.smarthome.scene.model.scene.SpecScene.DeviceExtra
import com.xiaomi.smarthome.scene.model.scene.SpecScene.PhoneExtra
import com.xiaomi.smarthome.scene.model.scene.SpecScene.SpecSceneItem
import com.xiaomi.smarthome.scene.model.scene.SpecScene.Trigger
import com.xiaomi.smarthome.scene.ui.list.MySceneFragmentkt
import com.xiaomi.smarthome.scene.ui.list.SceneListViewModel
import com.xiaomi.smarthome.scene.view.IconUtils
import com.xiaomi.smarthome.setting.PluginSetting
import com.xiaomi.smarthome.specscene.SpecSceneEditPageV2
import com.xiaomi.smarthome.specscene.SpecSceneEditPageV2.Companion.ACTION_SELECT_CONDITION
import com.xiaomi.smarthome.specscene.bean.ATplItem
import com.xiaomi.smarthome.specscene.bean.ScenePrivacy
import com.xiaomi.smarthome.specscene.bean.ui.DeviceActionUI.Companion.REQUEST_DEVICE_ACTION_CODE
import com.xiaomi.smarthome.specscene.bean.ui.SceneActionCard
import com.xiaomi.smarthome.specscene.device.datasource.SceneGroupDataSource.getAdminWearSupport
import com.xiaomi.smarthome.specscene.manager.SceneManagerV2
import com.xiaomi.smarthome.specscene.manager.SpecSceneConfigRepository
import com.xiaomi.smarthome.specscene.multicontrol.RemoteMSDataSource
import com.xiaomi.smarthome.specscene.personal.personstatus.PersonHomeSetting
import com.xiaomi.smarthome.specscene.personal.viewmodel.PersonalStatusManager
import com.xiaomi.smarthome.specscene.personal.viewmodel.PersonalStatusManager.findPersonItemByPersonId
import com.xiaomi.smarthome.specscene.personal.viewmodel.PersonalStatusManager.findPersonItemImgBySceneId
import com.xiaomi.smarthome.specscene.personal.viewmodel.PersonalStatusManager.oneItemShow
import com.xiaomi.smarthome.specscene.personal.viewmodel.QuickActionManager
import com.xiaomi.smarthome.specscene.personal.viewmodel.XiaoaiDeviceManager
import com.xiaomi.smarthome.stat.STAT
import com.xiaomi.smarthome.utils.dp
import com.xiaomi.youpin.login.okhttpApi.api.SystemAccountCompare
import io.reactivex.Observable
import miuix.core.util.RomUtils
import org.json.JSONArray
import org.json.JSONObject
import java.io.BufferedReader
import java.io.InputStreamReader
import java.lang.ref.WeakReference
import java.util.Locale
import java.util.TimeZone

class SpecSceneHelper {

    companion object {
        var shieldPhoneRecommendSwitch: Int = -1
        var isSupportGeofence: Boolean = false
        val recommendTplIconInfo: HashMap<String, JSONObject> = hashMapOf()

        init {
            MiJiaLog.onlyLogcat("jiangmd236", "SpecSceneHelper init")
            initGeofence()
        }

        fun launchRnPlugin(
            activity: WeakReference<Activity>,
            device: Device,
            sceneType: Int,
            id: Int,
            name: String,
            plugin_id: String?,
            command: String?,
            trId: Int?,
            value: Any?,
            oldValue: Any?,
            callback: ActivityResultCallback?,
            isAction: Boolean = false
//            onActivityResult: (activity: WeakReference<Activity>, resultCode: Int, requestCode: Int?, data: Intent?) -> Unit
        ) {
            val intent = Intent(plugin_id).apply {
                putExtra("action", command)
                //历史遗留问题：action和condition传的值不一样
                if (isAction) {
                    if (oldValue != null) {
                        if (oldValue is String) putExtra("value", oldValue)
                        else putExtra("value", oldValue.toString())
                    } else if (value != null) {
                        if (value is String) putExtra("value", value)
                        else putExtra("value", value.toString())
                    }
                } else {
                    if (value != null) {
                        if (value is String) putExtra("value", value)
                        else putExtra("value", value.toString())
                    }
                    if (oldValue != null) {
                        if (oldValue is String) putExtra("last_value", oldValue)
                        else putExtra("last_value", oldValue.toString())
                    }
                }
                putExtra("scene_type", sceneType)
                putExtra("actionId", id)
                putExtra("plug_id", plugin_id)

                // 和ios统一新增加参数传递给插件方
                putExtra("name", name)
                if (trId != null) {
                    putExtra("tr_id", trId.toString())
                }
            }

            ServiceApplication.getGlobalHandler().post {
                val realActivity = activity.get() ?: return@post
                val deviceInfo = CoreApi.getInstance().getPluginInfo(device.model)
                val dialog = XQProgressHorizontalDialog.build(
                    realActivity,
                    realActivity.getString(R.string.plugin_downloading) + (deviceInfo?.name ?: "")
                            + realActivity.getString(R.string.plugin)
                )
                val taskHolder = PluginDownloadTask()
                var isNotInstallAndNotDownload = false
                if (CoreApi.getInstance()
                        .getInstalledPackageInfo(device.model) == null && CoreApi.getInstance()
                        .getDownloadedPackageInfo(device.model) == null
                ) {
                    isNotInstallAndNotDownload = true
                }
                PluginApi.getInstance().sendMessage(
                    ServiceApplication.getAppContext(),
                    device.model,
                    IXmPluginMessageReceiver.MSG_GET_SCENE_VALUE,
                    intent,
                    DeviceRouterFactory.getDeviceWrapper().newDeviceStat(device),
                    null,
                    true,
                    object : SendMessageCallback() {
                        override fun onDownloadStart(record: String, task: PluginDownloadTask) {
                            task.copyTo(taskHolder)
                            val realActivity = activity.get() ?: return
                            if (realActivity.isFinishing) {
                                return
                            }
                            if (realActivity.isDestroyed) {
                                return
                            }
                            dialog?.setProgress(100, 0)
                            dialog?.hideProgressNumber()
                            dialog?.isCancelable = true
                            dialog?.show()
                            dialog?.setOnCancelListener {
                                CoreApi.getInstance().cancelPluginDownload(
                                    device.model, taskHolder, null
                                )
                            }

                        }

                        override fun onDownloadProgress(record: String?, percent: Float) {
                            if (isNotInstallAndNotDownload) {
                                var percentInt = (percent * 100).toInt()
                                if (percentInt >= 99) {
                                    percentInt = 99
                                }
                                dialog?.setProgress(100, percentInt)
                            } else {
                                dialog?.setProgress(100, (percent * 100).toInt())
                            }
                        }

                        override fun onDownloadSuccess(record: String?) {
                            if (!isNotInstallAndNotDownload) dialog?.dismiss()
                        }

                        override fun onDownloadFailure(error: PluginError?) {
                            if (!isNotInstallAndNotDownload) dialog?.dismiss()
                        }

                        override fun onDownloadCancel() {
                            if (!isNotInstallAndNotDownload) dialog?.dismiss()
                        }

                        override fun onSendSuccess(result: Bundle?) {
                            if (isNotInstallAndNotDownload) dialog?.dismiss()
                        }

                        override fun onSendFailure(error: Error?) {
                            if (isNotInstallAndNotDownload) dialog?.dismiss()
                        }

                        override fun onSendCancel() {
                            if (isNotInstallAndNotDownload) dialog?.dismiss()
                        }

                        override fun onMessageSuccess(result: Intent?) {
                            callback?.onCall(
                                activity,
                                Activity.RESULT_OK,
                                REQUEST_DEVICE_ACTION_CODE,
                                result
                            )
                        }

                        override fun onMessageFailure(errorCode: Int, errorInfo: String?) {
                            callback?.onCall(activity, Activity.RESULT_CANCELED, null, null)
                        }
                    }
                )
            }
        }

        fun getValueDisplayFromAction(at: ATplItem, tmpOneAction: SpecScene.Action?): String {
            //滚轴的情况：1.单位转换  2.摄氏度华氏度转换 3.spec设备的情况
            return ""
        }

        fun isPluginValue(deviceId: String, tplItemId: Int, paramName: String?): Boolean {
            //3.插件配置信息中获取，是否要进入插件
            val device = findDeviceById(deviceId)
            val model = device?.model
            val pluginRecord = model?.let { CoreApi.getInstance().getInstalledPackageInfo(it) }
            if (pluginRecord?.isRNPlugin == true) {
                if (DevelopSharePreManager.getInstance().isRNDebugEnableV2) {//开发折模式
                    //开发者模式下调试
                    val jsonObject = RnDebugFileUtil.getNativeDebugPluginInfo(model)
                    if (jsonObject != null) {
                        val isCheck =
                            jsonObject.optBoolean(RnDebugConstant.JSON_KEY_IS_CHECK, false)
                        val scenIdIsCheck =
                            jsonObject.optBoolean(RnDebugConstant.JSON_KEY_SCENE_ID_IS_CHECK, false)
                        val sceneId = jsonObject.optString(RnDebugConstant.JSON_KEY_SCENE_ID, "")
                        if (isCheck && scenIdIsCheck && sceneId.equals(tplItemId.toString())) {
                            // 只有调试的插件选中且调试场景选中且增加了调试的id
                            return true
                        }
                    }
                }
                // 如果是rn，按照配置来
                val triggers =
                    PluginSetting.getPluginProjectJsonConditions(pluginRecord.packagePath)
                if (triggers?.contains(tplItemId.toString()) == true) {
                    return true
                }
            }

            //4.开发者平台的扩展程序参数（plugin_id）
            if (!TextUtils.isEmpty(paramName) && device != null) {
                return true
            }
            return false
        }

        fun isNextDay(fromHour: Int, fromMinute: Int, toHour: Int, toMinute: Int): Boolean {
            return SpecScene.TimeWindow.isNextDay(fromHour, fromMinute, toHour, toMinute)
        }

        fun showDisplayTime(hour: Int?, minute: Int?): String {
            if (hour == null) {
                return ""
            }
            if (minute == null) {
                return ""
            }
            return SpecScene.TimeWindow.showDisplayTime(hour, minute)
        }

        fun getTimerParam(param: CorntabParam): String {
            var param = param
            if (param == null) {
                param = CorntabParam()
            }
            val buffer = StringBuilder()
            if (param.second == -1) buffer.append("* ") else buffer.append(param.second).append(" ")

            if (param.minute == -1) buffer.append("* ") else buffer.append(param.minute).append(" ")
            if (param.hour == -1) buffer.append("* ") else buffer.append(param.hour).append(" ")
            buffer.append("* ")
            buffer.append("* ")

            var begin = false
            var weekBuilder = StringBuilder()
            for (i in 0..6) {
                if (param.weeks[i]) {
                    if (begin) weekBuilder.append(",")
                    weekBuilder.append(i)
                    begin = true
                }
            }
            if (weekBuilder.isEmpty() || weekBuilder.length == 13) {
                buffer.append("*")
            } else {
                buffer.append(weekBuilder.toString())
            }

            buffer.append(" ").append("*")
            return buffer.toString()
        }

        fun getTimeDescAndRepeat(
            savedTimeValue: String?,
            savedTimeZone: String?
        ): Pair<String, String> {
            savedTimeValue ?: return Pair("", "")
            val split = savedTimeValue.split(" ")
            if (split.size != 7) return Pair("", "")
            var isOnceTimer = false
            var isDailyTimer = false
            val localCorntabParam = CorntabUtils.convertSceneCrontabByTimeZone(
                TimeZone.getDefault(),
                TimeZone.getTimeZone(savedTimeZone ?: "Asia/Shanghai"),
                CorntabParam().apply {
                    second = 0
                    minute = split[1].toInt()
                    hour = split[2].toInt()
                    isOnceTimer = !TextUtils.equals(split[3], "*") && !TextUtils.equals(
                        split[4],
                        "*"
                    )//兼容1.0转化过来的数据，年不判断了
                    if (isOnceTimer) {
                        day = split[3].toInt()
                        month = split[4].toInt()
                        year = if (!TextUtils.equals(split[6], "*")) {
                            split[6].toInt()
                        } else -1
                    } else {
                        day = -1
                        month = -1
                        year = -1
                    }
                    for (i in 0..6) {
                        weeks[i] = false
                    }
                    if (!filter.isNullOrEmpty() || TextUtils.equals(
                            "*",
                            split[5]
                        ) || split[5].trim().length == 13
                    ) {
                        for (i in 0..6) {
                            weeks[i] = false
                        }
                    } else {
                        val split5 = split[5].split(",")
                        for (str in split5) {
                            try {
                                weeks[str.toInt()] = true
                            } catch (e: Exception) {
                                e.printStackTrace()
                            }
                        }
                    }
                }
            )

            val repeatStr =
                if (isOnceTimer) CommonApplication.getAppContext().resources.getString(R.string.smarthome_scene_timer_once)
                else {
                    if (localCorntabParam.weeks == null || localCorntabParam.weeks.isEmpty() || hashSetOf<Boolean>().apply {
                            localCorntabParam.weeks.forEach {
                                add(
                                    it
                                )
                            }
                        }.size <= 1)
                        CommonApplication.getAppContext().resources.getString(R.string.smarthome_scene_timer_everyday)
                    else {
                        val dayList =
                            CommonApplication.getAppContext().resources.getStringArray(com.xiaomi.smarthome_scene_external.R.array.weekday_short)
                        var hasSunday = false
                        val builder = StringBuilder()
                        localCorntabParam.weeks.forEachIndexed { index, b ->
                            if (b) {
                                if (index == 0) hasSunday = true
                                else builder.append(dayList[index]).append(" ")
                            }
                        }
                        if (hasSunday) {
                            builder.append(dayList[0])
                        }
                        builder.toString().trim().replace(" ", ",")
                    }

                }
            val timeStr = showDisplayTime(localCorntabParam.hour, localCorntabParam.minute)
            return Pair(timeStr, repeatStr)
        }

        fun fillIcon(did: String, model: String, imgView: SimpleDraweeView, default: Int) {
            SmartHomeSceneUtility.setSceneDeviceIcon(imgView, did, model, default)
        }

        fun findDeviceById(did: String?): Device? {
            did ?: return null

            var result: Device? = SmartHomeDeviceManager.getInstance()
                .getDeviceByDid(did, DeviceFilter.FILTER_SPLITSUB)
            if (result == null) result = SmartHomeDeviceManager.getInstance().getSubDeviceByDid(did)
            if (result == null) result = SmartHomeDeviceManager.getInstance().findSoloDevices()
                .firstOrNull { d -> d.did == did }
            if (result == null) return null
            return DeviceFilter.FILTER_SPLITSUB.filter(result)
        }

        fun isDeletedDevice(d: Device?): Boolean {
            if (d == null) return true
            if (SmartHomeDeviceManager.isSoloPhone(d.model) || SmartHomeDeviceManager.isWatch(d.model)) {
                return !SmartHomeDeviceManager.getInstance().findSoloDevices()
                    .any { it.did.equals(d.did) }
            }
            if (isCarRoomDevice(d)) {
                //如果车上的临时设备，拿走之后算删除的话。。
//                return !d.isOnline && isFromCarTempDevice(SceneRuleCaculator.getCurrentCarRooms().getOrNull(0)?.id,d.did)
                return false
            }
            val home = HomeManager.getInstance().didHomeMap[d.did]
            return home == null ||
                    home.permitLevel < Home.PERMIT_HOME_MANAGER
        }

        fun isOtherHomeDevice(d: Device?, homeId: String?): Boolean {
            if (d == null) return true
            if (SmartHomeDeviceManager.isSoloPhone(d.model) || SmartHomeDeviceManager.isWatch(d.model)) return false
            //如果是车房间展示的设备，会在所有家庭下展示，因此永远不会展示成设备在其他家庭
            if (isCarRoomDevice(d)) return false
            return !TextUtils.equals(HomeManager.getInstance().getHomeByDid(d.did)?.id, homeId)

        }

        fun isOtherCarHomeDevice(
            d: Device?,
            homeId: String? = SceneRuleCaculator.getCurrentCarRoom()?.homeID
        ): Boolean {
            if (d == null) return true
            if (d.model.contains("phone") || d.model.contains("o62")) return false
            val carRoomDevices = getCarRoomByHomeId(homeId)?.getCarRoomDevices() ?: return false
            if (!carRoomDevices.devices.map { it.did }.contains(d.did)) return false
            if (carRoomDevices.notTemp.contains(d.did)) return false
            if (carRoomDevices.temp.contains(d.did) && d.isOnline) return false
            return true
        }

        fun isCarRoomDevice(d: Device?): Boolean {
            if (d == null) return false
            if (getCarRoomByDid(
                    d.did,
                    inDids = true,
                    inTemporaryDevice = true,
                    inCars = true
                ) != null
            ) return true
            return false
        }

        fun isCarRoom(rid: String?): Boolean {
            if (rid == null) return false
            val carRoom = CarManager.getCarRoomByRoomid(rid)
            return carRoom != null //&& carRoom.shareflag//是共驾人 不是 管理员家庭的情况
        }

        fun isCarHome(hid: String?): Boolean {
            if (hid == null) return false
            val carRoom = CarManager.getCarRoomByHomeId(hid)
            return carRoom != null //&& carRoom.shareflag//是共驾人 不是 管理员家庭的情况
        }

        fun isDeviceOnline(device: Device?): Boolean {
            if (device == null) return true
//            if (whichCarRoomTempDevice(device.did) != null && !device.isOnline) return false
            return device.isOnlineAdvance || device is BleDevice || CardRouterFactory.getCardManager()
                .getGridCardPair(device)?.card?.isLowPowerDevice == true
        }

        fun isEditableSceneAction(action: SpecScene.Action): Boolean {
            return when (action.type) {
                ConstantType.PAYLOAD_TYPE_EXECUTE_SCENE, ConstantType.PAYLOAD_TYPE_ON_OFF_SCENE -> {
//                    val sceneId = (action.payload as? SpecScene.Action.AutoScenePayload)?.sceneId
//                        ?: (action.payload as? SpecScene.Action.ClickScenePayload)?.sceneId
//                    if (sceneId == null) false
//                    SceneManagerV2.getInstance().getSceneById(sceneId) != null
                    action.nestInfo?.deleteMark == false
                }

                ConstantType.PAYLOAD_TYPE_RPC -> {
                    val device = SmartHomeDeviceManager.getInstance()
                        .getDeviceByDid((action.payload as? RpcPayload)?.did)
                    !(isDeletedDevice(device) || isOtherHomeDevice(
                        device,
                        HomeManager.getInstance().currentHomeId
                    ) || isOtherCarHomeDevice(device))
                }

                else -> true
            }
        }

        fun getRealCommand(command: String?, model: String?, saId: Int): String {
            if (command == null) return ""
            if (model == null || TextUtils.isEmpty(model)) return command

            val tmpCommand =
                if (command.startsWith(model)) command.replace("$model.", "").trim() else command
            if (!TextUtils.equals(tmpCommand.trim(), command.trim())) {
                return tmpCommand

            }
            //不符合model.command规则的
            val split = command.split(".")
            if (split != null) {
                return split.last()
            }
            STAT.click.split_command_error(saId, model, command)
            return command
        }

        fun getRealKey(
            key: String?,
            model: String,
            scId: Int,
            isReplace: Boolean = false,
            replaceKey: String? = null
        ): String {
            if (key == null) return ""
            var returnKey = key
            if (isReplace && !TextUtils.isEmpty(replaceKey)) {
                returnKey = replaceKey!!
            }
            val tmpKey = returnKey.replace("$model.", "").trim()
            if (!TextUtils.equals(tmpKey.trim(), key.trim())) {
                return tmpKey
            }
            val split = returnKey.split(".")
            if (split.isEmpty()) return key
            if (split.size == 5) return "${split[0]}.${split[4]}"
            if (split.size == 6) return "${split[0]}.${split[4]}.${split[5]}"
            STAT.click.split_key_error(scId, model, returnKey)
            return returnKey
        }

        fun getConfigKey(triggerKey: String, model: String): String? {
            val split = triggerKey.split(".")
            val builder = StringBuilder()
            if (split.isNotEmpty()) {
                builder.append(split[0]).append(" ").append(model).append(" ")
                for (i in 2 until split.size) {
                    builder.append(split[i]).append(" ")
                }
                return builder.toString().trim().replace(" ", ".")
            }
            return null
        }

        fun isReadConvertResult(homeId: String): Boolean {
            return SharePrefsManager.getSettingBoolean(
                ServiceApplication.getAppContext(),
                "convert_process_result_show",
                HomeManager.getInstance().currentHomeId,
                true
            )
        }

        fun setUnReadConvertResult(homeId: String) {
            SharePrefsManager.setSettingBoolean(
                ServiceApplication.getAppContext(),
                "convert_process_result_show",
                HomeManager.getInstance().currentHomeId,
                false
            )
        }

        /**
         * 数据分页，UI不能分页
         */
        fun getRecTplObservable(
            context: Context?,
            homeId: String,
            ownerId: Long,
            did: String?,
            isFromPlugin: Boolean
        ): Observable<out ArrayList<RecommendTpl>> {
            if (ServerCompact.isInternationalServer(context)) {
                return Observable.create { emitter -> emitter.onNext(arrayListOf()) }
            }
            val tmp: ArrayList<RecommendTpl> = arrayListOf()
            return RemoteApi.getTplPageandnext(
                context,
                homeId,
                ownerId,
                did,
                isFromPlugin,//did != null,
                1
            ).concatMap { t ->
                MiJiaLog.writeLogOnAll(LogType.SCENE, "recScene", "getTplList Success")
                tmp.addAll(t.results)
                Observable.fromArray(tmp)
            }
        }

        fun isUserClick(scene: SpecScene?): Boolean {
            var result = false
            scene?.triggers?.forEach { trigger ->
                if (TextUtils.equals(trigger.key, "user.click")) {
                    result = true
                    return@forEach
                }
            }
            return result
        }

        fun findOneSiteRelatedScene(
            tpl: RecommendTpl,
            site: RecommendTpl.SiteItem?,
            tplRelatedScenes: List<SpecScene>?,
            ignoreEnable: Boolean = false
        ): SpecScene? {
            if (site == null) return null
            var findOneScene: SpecScene? = null
            when (site.siteType) {
                1 -> {
                    findOneScene = tplRelatedScenes?.findLast {
                        (ignoreEnable || it.enable) && TextUtils.equals(
                            it.homeId,
                            site.siteId
                        ) && (TextUtils.isEmpty(
                            it.roomId
                        ) || TextUtils.equals("0", it.roomId))
                    }
                }

                2 -> {
                    findOneScene = tplRelatedScenes?.findLast {
                        (ignoreEnable || it.enable) && TextUtils.equals(
                            it.roomId,
                            site.siteId
                        )
                    }
                }

                3 -> {
                    findOneScene = tplRelatedScenes?.findLast {
                        (ignoreEnable || it.enable) &&
                                (tpl.trigger.singleDevice == true && it.triggers.any { trigger ->
                                    TextUtils.equals(
                                        trigger.src,
                                        "device"
                                    ) && TextUtils.equals(
                                        (trigger.extra as DeviceExtra).did,
                                        site.siteId
                                    )
                                })

                    }
                }

            }

            return findOneScene
        }

        fun isDeviceIn(did: String?, ss: SpecScene): Boolean {
            if (TextUtils.isEmpty(did)) return false
            if (TextUtils.isEmpty(ss.roomId) || TextUtils.equals("0", ss.roomId)) {
                for (i in 0 until (ss.actions?.size ?: 0)) {
                    if (ss.actions[i].type == 0 && TextUtils.equals(
                            (ss.actions[i].payload as RpcPayload).did,
                            did
                        )
                    ) {
                        return true
                    }
                }
            } else {
                HomeManager.getInstance().getRoomById(ss.roomId)?.also { room ->
                    room.dids?.forEach { _ ->
                        for (i in 0 until (ss.actions?.size ?: 0)) {
                            if (ss.actions[i].type == 0 && TextUtils.equals(
                                    (ss.actions[i].payload as RpcPayload).did,
                                    did
                                )
                            ) {
                                return true
                            }
                        }
                    }
                }

            }
            return false
        }

        fun isDeviceInSites(did: String?, siteList: List<RecommendTpl.SiteItem>?): Boolean {
            val roomId = HomeManager.getInstance().getRoomByDid(did)?.id
            for (i in 0 until (siteList?.size ?: 0)) {
                if (siteList?.get(i)?.siteType ?: -1 == 2 && TextUtils.equals(
                        roomId,
                        siteList?.get(i)?.siteId
                    )
                ) {
                    return true
                }
            }
            return false
        }

        fun getState(s: SpecScene?): Int {
            if (s == null || TextUtils.isEmpty(s.id) || TextUtils.equals("0", s.id)) return 1
            if (s.enable) return 2
            return 3
        }

        fun getRecSceneDisplayName(s: SpecScene?, t: RecommendTpl?): String {
            if (s == null) {
                return ""
            }
            if (s.recommId == 0L) {//自定义手动场景
                return s.name
            }
            if (s.recommId == ConstantType.TPL_ID_OPEN_NO_DISTURB) {//打开勿扰特殊处理
                return CommonApplication.getAppContext()
                    .getString(R.string.scene_name_open_nodisturb)
            }
            if (s.recommId == ConstantType.TPL_ID_CLOSE_NO_DISTURB) {//关闭勿扰特殊处理
                return CommonApplication.getAppContext()
                    .getString(R.string.scene_name_close_nodisturb)
            }
            t?.also {
                if (it.templateType == 0) {//普通推荐场景
                    return it.name ?: s.name
                }
                if (it.recommendStrategy > 0) {//生活场景、特殊场景
                    if (!TextUtils.isEmpty(s.roomId) && !TextUtils.equals("0", s.roomId)) {//全屋维度
                        HomeManager.getInstance().getRoomById(s.roomId)?.also { r ->
                            return CommonApplication.getAppContext()
                                .getString(R.string.prefix_rec_room_scene_name, r.name, t.name)
                        }
                        return t.name ?: s.name
                    } else {
                        return CommonApplication.getAppContext()
                            .getString(R.string.prefix_rec_whole_house_scene_name, t.name)
                    }
                }
                return it.name ?: s.name
            }
            return s.name
        }

        fun isLogin(): Boolean {
            return ServiceApplication.getStateNotifier().currentState == AppStateNotifier.STATE_LOGIN_SUCCESS
        }

        fun getTitleOfAction(type: Int, action: SpecScene.Action): String {

            return when (type) {
                ConstantType.PAYLOAD_TYPE_RPC -> if (action.from == 3) CommonApplication.getAppContext()
                    .getString(R.string.action_cat_on_off_device) else findDeviceById((action.payload as? RpcPayload)?.did)?.name
                    ?: (action.payload as? RpcPayload)?.deviceName ?: ""

                ConstantType.PAYLOAD_TYPE_EXECUTE_SCENE,
                ConstantType.PAYLOAD_TYPE_ON_OFF_SCENE -> action.name

                ConstantType.PAYLOAD_TYPE_DELAY -> CommonApplication.getAppContext()
                    .getString(R.string.action_push_sub)

                ConstantType.PAYLOAD_TYPE_PUSH -> CommonApplication.getAppContext()
                    .getString(R.string.action_push_sub)

                else -> ""
            }
        }

        fun getActionContentOfActions(
            type: Int,
            actions: ArrayList<SpecScene.Action>?
        ): String {

            if (actions?.isEmpty() == true) CommonApplication.getAppContext()
                .getString(R.string.scene_param_not_set)
            return when (type) {
                ConstantType.PAYLOAD_TYPE_PUSH -> CommonApplication.getAppContext()
                    .getString(
                        R.string.action_cat_push_formatter,
                        (actions?.getOrNull(0)?.payload as? SpecScene.Action.PushPayload)?.desc
                            ?: ""
                    )

                ConstantType.PAYLOAD_TYPE_ON_OFF_SCENE -> CommonApplication.getAppContext()
                    .getString(if ((actions?.getOrNull(0)?.payload as? SpecScene.Action.AutoScenePayload)?.enable == true) R.string.scene2_open else R.string.scene2_close)

                ConstantType.PAYLOAD_TYPE_EXECUTE_SCENE -> CommonApplication.getAppContext()
                    .getString(R.string.scene2_to_execute)

                ConstantType.PAYLOAD_TYPE_RPC -> {
                    if (actions?.getOrNull(0)?.from == 0) {
                        CommonApplication.getAppContext()
                            .getString(R.string.scene_param_not_set)
                    } else if (actions?.getOrNull(0)?.from == 1) {
                        if (actions.size == 1) actions[0].name
                            ?: "" else CommonApplication.getAppContext().resources.getQuantityString(
                            R.plurals.scene_action_content_formatter,
                            actions.size,
                            actions.size,
                            actions[0].name ?: ""
                        )
                    } else if (actions?.getOrNull(0)?.from == 3) {
                        var actionNameKey = if (actions.isEmpty()) CommonApplication.getAppContext()
                            .getString(R.string.scene_param_not_set) else ""
                        RemoteMSDataSource.msConfigData.aGroups.values.forEach { gb ->
                            val find =
                                gb.options?.firstOrNull { option -> actions[0].tcaId == option.id }
                            if (find != null) {
                                actionNameKey =
                                    RemoteMSDataSource.msConfigData.rules[find.detail[0].key_rule]?.action_desc
                                        ?: ""
                                return@forEach
                            }
                        }

                        RemoteMSDataSource.msLanData.lanStrMap[actionNameKey]
                            ?: ""
                    } else {
                        ""
                    }
                }

                else -> ""
            }
        }

        fun getIconOfActions(actions: ArrayList<SpecScene.Action>?): List<String> {
            return actions?.map {
                getIconOfAction(it.type, it)
            } ?: arrayListOf()
        }

        fun getIconOfAction(
            type: Int,
            action: SpecScene.Action?,
        ): String {
            return when (type) {
                ConstantType.PAYLOAD_TYPE_RPC, 4 -> {
                    var iconUrl: String? = ""
//                    val tmpIcon = DeviceFactory.getPluginIcon((action?.payload as? SpecScene.Action.RpcPayload)?.model)
                    try {

                        if (action?.from == 3) {
                            val de = action.payload as? RpcPayload
                            if (de?.model?.contains(".switch.") == true || TextUtils.equals(
                                    de?.model,
                                    "lumi.ctrl_neutral2.v1"
                                )
                                || TextUtils.equals(de?.model, "lumi.ctrl_ln2.v1")
                                || TextUtils.equals(de?.model, "lumi.ctrl_ln2.aq1")
                            ) {
                                val rd = findDeviceById(de?.did)
                                if (de != null && rd != null) {
                                    val subDev = SmartHomeDeviceManager.getInstance()
                                        .getDeviceByDid(
                                            de.did + ".s" + ((de.value as? JSONArray)?.optJSONObject(
                                                0
                                            )
                                                ?.optInt("siid") ?: "")
                                        )
                                    if (subDev != null) {
                                        iconUrl = subDev.icon
                                    }
                                }


                            }
                        }
                    } catch (e: Exception) {
                        iconUrl = ""
                    }
                    if (TextUtils.isEmpty(iconUrl) || iconUrl?.startsWith("https://") == false) {
                        iconUrl = SmartHomeSceneUtility.getSceneDeviceIconUrlBy(
                            (action?.payload as? RpcPayload)?.did,
                            (action?.payload as? RpcPayload)?.model
                        )
                    }

                    if (TextUtils.isEmpty(iconUrl)) getResouceUriString(R.drawable.device_list_phone_no) else iconUrl
                        ?: getResouceUriString(R.drawable.device_list_phone_no)
                    if (action?.payload is QuickActionPayload)
                        if (ConstantType.LIGHT_TYPES.contains((action.payload as QuickActionPayload).value.command_type))
                            getResouceUriString(R.drawable.quick_action_light)
                        else getResouceUriString(R.drawable.quick_action_curtain)
                    else iconUrl ?: getResouceUriString(R.drawable.device_list_phone_no)
                }

                ConstantType.PAYLOAD_TYPE_EXECUTE_SCENE -> {
                    return action?.nestInfo?.sceneIcon ?: ConstantType.USER_CLICK_ICON_URL
                }

                ConstantType.PAYLOAD_TYPE_ON_OFF_SCENE -> {
                    val icon = action?.nestInfo?.sceneIcon
                    return if (icon?.isNotBlank() == true) icon else getResouceUriString(R.drawable.icon_add_toast_auto)
                }

                ConstantType.PAYLOAD_TYPE_DELAY -> getResouceUriString(R.drawable.icon_add_toast_delay)
                ConstantType.PAYLOAD_TYPE_PUSH -> getResouceUriString(R.drawable.std_scene_icon_push)

                else -> ""
            }

        }

        fun getIconOfTrigger(trigger: Trigger?): String = IconUtils.getIconOfTrigger(
            trigger?.src,
            trigger?.key,
            (trigger?.extra as? DeviceExtra)?.did,
            (trigger?.extra as? DeviceExtra)?.model,
            trigger,
            if (TextUtils.equals(
                    ConstantType.TriggerConditionType.PHONE.key,
                    trigger?.src
                )
            ) (trigger?.extra as? PhoneExtra)?.typeId else null
        )


        fun isUnsetActionCard(actions: ArrayList<SpecScene.Action>?): Boolean {
            if (actions?.isEmpty() == true) return true
            actions?.forEach {
                if (it.type == ConstantType.PAYLOAD_TYPE_RPC && (it.from == 0 || TextUtils.isEmpty(
                        it.name
                    ))
                ) return true
                if (it.payload == null) return true
                if (it.type == ConstantType.PAYLOAD_TYPE_ON_OFF_SCENE && (it.payload as? SpecScene.Action.AutoScenePayload)?.enable == null)
                    return true
                if(it.type == ConstantType.PAYLOAD_QUICK_ACTION && notHasQuickActionDevice() ) return true
            }
            return false
        }

        fun isUnsetAction(action: SpecScene.Action?): Boolean {
            if (action == null) return true
            if (action.type == ConstantType.PAYLOAD_TYPE_RPC && (action.from == 0 || TextUtils.isEmpty(
                    action.name
                ))
            ) return true
            if (action.payload == null) return true
            if (action.type == ConstantType.PAYLOAD_TYPE_ON_OFF_SCENE && (action.payload as? SpecScene.Action.AutoScenePayload)?.enable == null)
                return true
            return false
        }

        fun isUnsetTriggerCard(trigger: SpecScene.Trigger?): Boolean {
            if (trigger == null) return true
            if (trigger.extra == null) {
                return if (TextUtils.equals("timer", trigger.src)) {
                    trigger.valueJson == null
                } else true
            }
            if (trigger.extra is DeviceExtra && TextUtils.isEmpty(trigger.key)) {
                return true
            }
            if (trigger.extra is SpecScene.CloudExtra && TextUtils.isEmpty((trigger.extra as SpecScene.CloudExtra).address)) {
                return true
            }
            return false
        }

        fun isUnsetConditionCard(condition: SpecScene.Condition?): Boolean {
            if (condition == null) return true
            if (condition.extra == null) {
                return if (TextUtils.equals("timer", condition.src)) {
                    condition.valueJson == null
                } else true
            }
            if (condition.extra is DeviceExtra && TextUtils.isEmpty(condition.key)) {
                return true
            }
            if (condition.extra is SpecScene.CloudExtra && TextUtils.isEmpty((condition.extra as SpecScene.CloudExtra).address)) {
                return true
            }
            return false
        }

        fun getResouceUriString(resource: Int): String = IconUtils.getResouceUriString(resource)

        fun getActionStateDescription(actions: ArrayList<SpecScene.Action>): String? {
            if (actions.size == 0) return ""
            when (actions[0].type) {
                ConstantType.PAYLOAD_TYPE_RPC ->
                    if (actions[0].from == 3) {
                        if (actions.size == 1) {
                            val d =
                                findDeviceById((actions[0].payload as? RpcPayload)?.did)
                            if (d == null) return CommonApplication.getAppContext()
                                .getString(R.string.scene_deleted_simple)
                            else if (!TextUtils.equals(
                                    HomeManager.getInstance().didHomeMap[d.did]?.id,
                                    HomeManager.getInstance().currentHomeId
                                )
                            ) {
                                return CommonApplication.getAppContext()
                                    .getString(R.string.device_other_home_simple)
                            }
                        } else {
                            if (actions.any {
                                    val d =
                                        findDeviceById((it.payload as? RpcPayload)?.did)
                                    d == null || !TextUtils.equals(
                                        HomeManager.getInstance().didHomeMap[d.did]?.id,
                                        HomeManager.getInstance().currentHomeId
                                    )
                                })
                            //TODO:问产品  应该提示部分删除 或者 有删除
                                return CommonApplication.getAppContext()
                                    .getString(R.string.scene_deleted_simple)
                        }
                    } else if (actions[0].from == 1) {
                        val d =
                            findDeviceById((actions[0].payload as? RpcPayload)?.did)
                        if (d == null) return CommonApplication.getAppContext()
                            .getString(R.string.scene_deleted_simple)
                        else if (!TextUtils.equals(
                                HomeManager.getInstance().didHomeMap[d.did]?.id,
                                HomeManager.getInstance().currentHomeId
                            )
                        ) {
                            return CommonApplication.getAppContext()
                                .getString(R.string.device_other_home_simple)
                        } else if (!isOnlineInDisplay(d)) {
                            return "#${
                                CommonApplication.getAppContext()
                                    .getString(R.string.offline_device)
                            }"
                        }
                    }


                ConstantType.PAYLOAD_TYPE_EXECUTE_SCENE ->
//                    if (SceneManagerV2.getInstance().getSceneById(
//                            (actions[0].payload as? SpecScene.Action.ClickScenePayload)?.sceneId
//                                ?: ""
//                        ) == null
//                    )
                    if (actions[0].nestInfo?.deleteMark == true)
                        return CommonApplication.getAppContext()
                            .getString(R.string.scene_deleted_simple)

                ConstantType.PAYLOAD_TYPE_ON_OFF_SCENE ->
//                    if (SceneManagerV2.getInstance()
//                            .getSceneById(
//                                (actions[0].payload as? SpecScene.Action.AutoScenePayload)?.sceneId
//                                    ?: ""
//                            ) == null
//                    )
                    if (actions[0].nestInfo?.deleteMark == true)
                        return CommonApplication.getAppContext()
                            .getString(R.string.scene_deleted_simple)

                else -> SceneActionCard.TYPE.UNKNOWN
            }
            return null
        }

        fun getActionState(action: SpecScene.Action): String? {
            when (action.type) {
                ConstantType.PAYLOAD_TYPE_RPC ->
                    if (action.from == 3) {
                        val d =
                            findDeviceById((action.payload as? RpcPayload)?.did)
                        if (d == null) return CommonApplication.getAppContext()
                            .getString(R.string.scene_deleted_simple)
                        else if (!TextUtils.equals(
                                HomeManager.getInstance().didHomeMap[d.did]?.id,
                                HomeManager.getInstance().currentHomeId
                            )
                        ) {
                            return CommonApplication.getAppContext()
                                .getString(R.string.device_other_home_simple)
                        }

                    } else if (action.from == 1) {
                        val d =
                            findDeviceById((action.payload as? RpcPayload)?.did)
                        if (d == null) return CommonApplication.getAppContext()
                            .getString(R.string.scene_deleted_simple)
                        else if (!TextUtils.equals(
                                HomeManager.getInstance().didHomeMap[d.did]?.id,
                                HomeManager.getInstance().currentHomeId
                            )
                        ) {
                            return CommonApplication.getAppContext()
                                .getString(R.string.device_other_home_simple)
                        } else if (!isOnlineInDisplay(d)) {
                            return "#${
                                CommonApplication.getAppContext()
                                    .getString(R.string.offline_device)
                            }"
                        }
                    }


                ConstantType.PAYLOAD_TYPE_EXECUTE_SCENE ->
//                    if (SceneManagerV2.getInstance().getSceneById(
//                            (action.payload as? SpecScene.Action.ClickScenePayload)?.sceneId
//                                ?: ""
//                        ) == null
//                    )
                    if (action.nestInfo?.deleteMark == true)
                        return CommonApplication.getAppContext()
                            .getString(R.string.scene_deleted_simple)

                ConstantType.PAYLOAD_TYPE_ON_OFF_SCENE ->
//                    if (SceneManagerV2.getInstance()
//                            .getSceneById(
//                                (action.payload as? SpecScene.Action.AutoScenePayload)?.sceneId
//                                    ?: ""
//                            ) == null
//                    )
                    if (action.nestInfo?.deleteMark == true)
                        return CommonApplication.getAppContext()
                            .getString(R.string.scene_deleted_simple)

                else -> SceneActionCard.TYPE.UNKNOWN
            }
            return null
        }

        fun hasOldGateWay(): Boolean {
            val oldGatewayModels = setOf(
                "lumi.gateway.v3",
                "lumi.acpartner.v1",
                "lumi.camera.v1",
                "lumi.acpartner.v2",
                "lumi.camera.aq1",
                "lumi.acpartner.v3",
                "lumi.gateway.mitw01",
                "lumi.gateway.mihk01",
                "lumi.gateway.mieu01",
                "lumi.camera.gwagl01",
                "lumi.gateway.lmuk01",
                "lumi.gateway.aqhm01",
                "lumi.gateway.aqhm02"

            )
            return HomeManager.getInstance()
                .getDeviceByHomeId(HomeManager.getInstance().currentHomeId)
                .any { it.isOwner && oldGatewayModels.contains(it.model) }
//            return true
        }

        fun isOnlineInDisplay(d: Device?): Boolean {
            if (d == null) return false
            val card = CardRouterFactory.getCardManager().getGridCardPair(d)
            return d.isOnlineAdvance || DeviceUtils.isCarDevice(d.model) || d is BleDevice || card != null && card.card.isLowPowerDevice
        }

        fun isBleNotConnet(d: Device?): Boolean {
            return !IMainPageApiProvider.getMainPageApi().isBlePropValid(d)
//            return d != null && d is BleDevice && (d as? BleDevice)?.isConnected == false
        }

        fun showMemberHint(
            context: FragmentActivity?,
            title: Int? = null,
            msg: Int? = null,
            onPositive: (() -> Unit)? = null,
            positiveText: Int? = null,
            negaticeText: Int? = null,
            onNegative: (() -> Unit)? = null
        ): MLAlertDialog? {
            context ?: return null
            val titleStr = title?.let { context.resources.getString(it) } ?: ""
            val dialog = MJMessageDialog.Builder(
                context,
                context.resources.getString(msg ?: R.string.member_scene_setting_hint),
                gravity = Gravity.CENTER
                )
                .onPositiveClick(
                    context.resources.getString(positiveText ?: R.string.mj_i_know)
                ) {
                    onPositive?.invoke()
                }
                .setCancelable(false)
                .setTitle(titleStr)
            onNegative?.also {
                dialog.onNegativeClick(
                    context.resources.getString(negaticeText ?: R.string.sh_common_cancel)
                ) {
                    it.invoke()
                }
            }
            return dialog.show()
        }

        fun getSpecSceneFromMainPage(
            isAuto: Boolean,
            scenes: List<CommonUsedScene>?,
            dids: List<String>?,
            roomId: String?
        ): SpecScene = SpecScene().apply {
            val curHome = HomeManager.getInstance().currentHome
            homeId = HomeManager.getInstance().currentHomeId
            timeWindow = SpecScene.TimeWindow.getDefault()
            enable = true
            if (roomId == null || TextUtils.equals(HomeManager.ROOMID_COMMON, roomId)) {
                isCommonUse = true
            } else if (!TextUtils.equals(HomeManager.ROOMID_DEFAULT, roomId)) {
                isCommonUse = false
                commonUsedRoomIds = arrayListOf<String>().apply { add(roomId) }
            }
            actions = arrayListOf()
            conditions = arrayListOf()
            triggers = arrayListOf()
            triggerExpress = 1//任一
            conditionExpress = 0//任一
            dids?.forEach {
                var device = SmartHomeDeviceManager.getInstance().getDeviceByDid(it)
                val tmpParentDid = DeviceUtils.getParentId(device)
                if (!TextUtils.isEmpty(tmpParentDid)) {
                    device = findDeviceById(tmpParentDid)
                }
                if (SmartHomeDeviceManager.isWatch(device.model) && curHome?.isShareManager == true) {
                    if(!getAdminWearSupport()) {
                        device = null
                    }
                }
                if (device != null) {
                    val tmpActions = arrayListOf<SpecScene.Action>()
                    val aTplItems =
                        SpecSceneConfigRepository.actionsDid[it]?.map { SpecSceneConfigRepository.actionsOnline[it] }
                    if (aTplItems?.isNotEmpty() == true) {
                        val aItem = aTplItems.getOrNull(0)
//                            for (aItem in aTplItems) {
                        val isFillInAction = aItem != null &&
                                aItem.rank != ConstantType.DEFAULT_TPL_RANK && aItem.mGroupId <= 0 && TextUtils.isEmpty(
                            aItem.mParamAction
                        ) && aItem.mAttr == null
                        if (aItem != null && isFillInAction) {
                            tmpActions.add(SpecScene.Action().apply {
                                type = ConstantType.PAYLOAD_TYPE_RPC
                                from = 1
                                tcaId = aItem.id
                                protocolType = aItem.protocolType ?: 2
                                name = aItem.mName
                                payload = RpcPayload().apply {
                                    did = device.did
                                    model = device.model
                                    deviceName = device.name
                                    deviceType = RpcPayload.getDeviecType(device.model)
                                    ownerUid = device.ownerId
                                    method = getRealCommand(
                                        aItem.mCommand,
                                        device.model,
                                        aItem.id
                                    )
                                    value = aItem.mValue
                                }
                            })
                        }
//                            }
                        if (tmpActions.isEmpty()) {
                            actions.add(SpecScene.Action().apply {
                                type = ConstantType.PAYLOAD_TYPE_RPC
                                from = 1
                                payload = RpcPayload().apply {
                                    did = device.did
                                    model = device.model
                                    deviceName = device.name
                                }
                            })
                        } else {
                            actions.addAll(tmpActions)
                        }

                    } else if (isAuto) {
                        val tTplItems =
                            SpecSceneConfigRepository.triggersDid[it]?.map { SpecSceneConfigRepository.triggersOnline[it] }
                        if (tTplItems?.isNotEmpty() == true) {
                            val tItem = tTplItems.getOrNull(0)
                            val isFillInTrigger = tItem != null
                                    && tItem.rank != ConstantType.DEFAULT_TPL_RANK
                                    && tItem.mGroupId <= 0
                                    && TextUtils.isEmpty(tItem.mParamAction)
                                    && tItem.mAttr == null
                            triggers.add(SpecScene.Trigger().apply {
                                if (isFillInTrigger) {
                                    valueType = if (isPluginValue(
                                            device.did,
                                            tTplItems.getOrNull(0)?.id ?: 0,
                                            tTplItems.getOrNull(0)?.mParamAction
                                        )
                                    ) 99 else tTplItems.getOrNull(0)?.valueType
                                        ?: 5
                                    key = getRealKey(
                                        tTplItems.getOrNull(0)?.mKey,
                                        device.model,
                                        tcaId
                                    )
                                    protocolType = tTplItems.getOrNull(0)?.protocolType ?: 2
                                    valueJson = tTplItems.getOrNull(0)?.mValue
                                    name = tTplItems.getOrNull(0)?.mName
                                    tcaId = tTplItems.getOrNull(0)?.id ?: 0
                                }
                                src = "device"
                                from = 1
                                extra = DeviceExtra(device)
                            })
                        }

                    }
                }
            }
            scenes?.forEach {
                actions.add(SpecScene.Action().apply {
                    type = ConstantType.PAYLOAD_TYPE_EXECUTE_SCENE
                    from = 1
                    payload = SpecScene.Action.ClickScenePayload().apply { sceneId = it.sceneId }
                    nestInfo = SpecScene.NestSceneSnapShot().apply {
                        deleteMark = false
                        sceneName = it.sceneName ?: ""
                        sceneIcon = it.iconUrl ?: ConstantType.USER_CLICK_ICON_URL
                    }
                })
            }


        }

        fun getGoEditPageIntent(
            context: Context,
            isAuto: Boolean,
            roomId: String?,
            batchFrom: Int
        ): Intent = Intent(context, SpecSceneEditPageV2::class.java).apply {
            putExtra("type", if (isAuto) "auto" else "click")
            putExtra("batch_from", batchFrom)
            putExtra("request_code", MySceneFragmentkt.CREATE_SCENE)


            if (!TextUtils.isEmpty(roomId)) {
                CarManager.getCarRoomByRoomid(roomId)?.also {
                    putExtra("car_home_id", it.homeID)
                }
                CarManager.getCarRoomByRoomid(roomId)?.also {
                    putExtra("car_home_id", it.homeID)
                }

                when {
                    TextUtils.equals(roomId, HomeManager.ROOMID_COMMON) -> {
                        putExtra("from_page_room", ConstantType.FROM_PAGE_ROOM_COMMON)
                    }

                    TextUtils.equals(roomId, HomeManager.ROOMID_DEFAULT) -> {
                        putExtra("from_page_room", ConstantType.FROM_PAGE_ROOM_DEFAULT)
                    }

                    else -> {
                        try {
                            putExtra("from_page_room", roomId)
                        } catch (e: Exception) {
                            putExtra("from_page_room", ConstantType.FROM_PAGE_ROOM_NONE)
                        }
                    }
                }
            } else {
                putExtra("from_page_room", ConstantType.FROM_PAGE_ROOM_NONE)
            }
        }

        fun getGoOldEditPageIntent(
            context: Context,
            requestCode: Int? = MySceneFragmentkt.CREATE_SCENE,
            did: String?,
            batchFrom: Int?,
        ): Intent = Intent(context, SmarthomeCreateAutoSceneActivity::class.java).apply {
            putExtra("request_code", requestCode)
            batchFrom?.also { putExtra("batch_from", it) }
            if (batchFrom == ConstantType.BATCH_FROM_PLUGIN) {
                putExtra(
                    SmarthomeCreateAutoSceneActivity.FROM,
                    SmarthomeCreateAutoSceneActivity.FROM_PLUGIN
                )
                did?.also { putExtra(SmarthomeCreateAutoSceneActivity.EXTRA_DEVICE_ID, it) }
            }
        }

        fun tryActiveABTestByPath(owner: LifecycleOwner, context: Context?, paths: List<String>?) {
            paths?.forEach {
                ABTesting.getConfigChangeLiveData(it).observe(owner) { config: ABConfig? ->
                    if (config != null && context != null) {
                        ABTesting.activeABTestByPath(context, config.expPath)
                    }
                }
            }

        }

        fun isContainCarItem(s: SpecScene): CarRoomData? {
            var result: CarRoomData? = null
            val triggerContainCar = s.triggers?.find {
                result = getCarRoomByDid(
                    (it.extra as? SpecScene.DeviceExtra)?.did,
                    inDids = true,
                    inCars = true,
                    inTemporaryDevice = true
                )
                result != null
            }
            if (triggerContainCar != null) return result
            val conditionContainCar = s.conditions?.find {
                result = getCarRoomByDid(
                    (it.extra as? SpecScene.DeviceExtra)?.did,
                    inDids = true,
                    inCars = true,
                    inTemporaryDevice = true
                )
                result != null
            }
            if (conditionContainCar != null) return result
            val actionContainCar = s.actions?.find {

                result = getCarRoomByDid(
                    (it.payload as? RpcPayload)?.did,
                    inDids = true,
                    inCars = true,
                    inTemporaryDevice = true
                )
                result != null
            }
            if (actionContainCar != null) return result
            return null
        }

        fun isShowLog(homeId: String): Boolean {
            if (!TextUtils.equals(HomeManager.getInstance().currentHomeId, homeId)) {
                if (CarManager.getCarRoomByHomeId(homeId) != null) return true

            }
            return TextUtils.equals(HomeManager.getInstance().currentHomeId, homeId)
        }

        fun isHyperOs2():Boolean{
            return try {
                RomUtils.isHyperOsRom() && RomUtils.getHyperOsVersion() >= 2
            } catch (e: Exception) {
                false
            }
        }
        fun fromWhichCarTempDevice(roomId: String?, did: String?): CarRoomData? {
            if (did == null) return null
            val tmpedRoom = whichCarRoomTempDevice(did).map { it.id }.toSet()
            if (tmpedRoom.contains(roomId)) {
                //临时设备
                val bindHome = getCarRoomByDid(
                    did,
                    inDids = true,
                    inCars = true,
                    inTemporaryDevice = false
                )
                if (bindHome != null) {
                    //如果临时设备是绑在车上的设备
                    return bindHome
                }
            }
            return null
        }

        fun isFromCarTempDevice(roomId: String?, did: String?): Boolean =
            fromWhichCarTempDevice(roomId, did) != null


        fun sceneCheckUserGray(
            context: Context,
            callback: AsyncCallback<JSONObject, Error>? = null
        ) {
            val md5uid = MD5Util.getMd5Digest(
                CoreApi.getInstance().miId
            )
            val mPrefs = context.getSharedPreferences(md5uid, Context.MODE_PRIVATE)
            getSceneUserGrayFromCache(context)

            val params = ArrayList<String>()
            params.add("scenev1_to_v2_switch")
            params.add("laboratory_scene2_switch")
            params.add("phone_scene_switch")
            params.add("shield_phone_recommend_switch")
            if(!CoreApi.getInstance().isInternationalServer()){
                params.add("perception_empty_display_type")
            }
            RemoteApi.sceneCheckUserGray(context, params,
                object : AsyncCallback<JSONObject, Error>() {
                    override fun onSuccess(result: JSONObject?) {
                        MiJiaLog.writeLogOnAll(
                            LogType.SCENE,
                            "grey check",
                            "sceneCheckUserGray from server ${result?.toString() ?: ""}"
                        )
                        SceneManagerV2.scenConvertFlag =
                            if (result?.optBoolean("scenev1_to_v2_switch") == true && ServerCompact.getCurrentServer(
                                    CommonApplication.getAppContext()
                                )?.let { isCurrentServerSupportSpecScene(it) } == true
                            ) 1 else 0
                        SceneManagerV2.specSceneEntranceFlag =
                            if (result?.optBoolean("laboratory_scene2_switch") == true && ServerCompact.getCurrentServer(
                                    CommonApplication.getAppContext()
                                )?.let { isCurrentServerSupportSpecScene(it) } == true
                            ) 1 else 0
                        SceneManagerV2.scenePhoneFlag =
                            if (result?.optBoolean("phone_scene_switch") == true) 1 else 0
                        shieldPhoneRecommendSwitch =
                            if (result?.optBoolean("shield_phone_recommend_switch") == true) 1 else 0
                        val hideOrGrey =
                            result?.optBoolean("perception_empty_display_type") ?: false //针对感知"回家"下不支持任何感知能力，是置灰，还是不展示，期望实验室有个切换开关 true 展示 false 置灰

                        if (!TextUtils.isEmpty(md5uid)) {

                            mPrefs.edit()
                                .putInt("scenev1_to_v2_switch", SceneManagerV2.scenConvertFlag)
                                .apply()
                            mPrefs.edit().putInt(
                                "laboratory_scene2_switch",
                                SceneManagerV2.specSceneEntranceFlag
                            ).apply()
                            mPrefs.edit().putInt(
                                "phone_scene_switch",
                                SceneManagerV2.scenePhoneFlag
                            ).apply()
                            mPrefs.edit().putInt(
                                "shield_phone_recommend_switch",
                                shieldPhoneRecommendSwitch
                            ).apply()
                            mPrefs.edit().putBoolean(
                                KEY_CUSTOM_HIDE_OR_GRAY,
                                hideOrGrey
                            ).apply()
                            // ConstantType.CONFIG_SCENE_GLOBAL_SETTING,
                            //                        ConstantType.KEY_CUSTOM_HIDE_OR_GRAY,
                        }
                        callback?.onSuccess(result)

                    }

                    override fun onFailure(error: Error?) {
                        MiJiaLog.writeLogOnAll(
                            LogType.SCENE,
                            "grey check",
                            "sceneCheckUserGray error ${error?.toString() ?: ""}"
                        )
                        callback?.onFailure(error)
                    }

                }
            )
        }

        fun isCurrentServerSupportSpecScene(serverBean: ServerBean): Boolean {
//            if(BuildConfig.DEBUG){
//                return ServerCompact.isStaging(serverBean) || ServerCompact.isSingaporeServer(serverBean) || ServerCompact.isIndiaServer(serverBean) || ServerCompact.isRussia(serverBean) || ServerCompact.isAmericaServer(serverBean) || ServerCompact.isChinaMainLand(serverBean)
//            }
//            return ServerCompact.isSingaporeServer(serverBean) || ServerCompact.isIndiaServer(serverBean) || ServerCompact.isRussia(serverBean) || ServerCompact.isAmericaServer(serverBean) || ServerCompact.isChinaMainLand(serverBean)
            return true
        }

        private fun getSceneUserGrayFromCache(context: Context) {
            val md5uid = MD5Util.getMd5Digest(
                CoreApi.getInstance().miId
            )
            if (!TextUtils.isEmpty(md5uid)) {
                val mPrefs = context.getSharedPreferences(md5uid, Context.MODE_PRIVATE)
                SceneManagerV2.scenConvertFlag =
                    if (ServerCompact.getCurrentServer(CommonApplication.getAppContext())
                            ?.let { isCurrentServerSupportSpecScene(it) } == true
                    ) mPrefs.getInt("scenev1_to_v2_switch", -1) else 0
                SceneManagerV2.specSceneEntranceFlag =
                    if (ServerCompact.getCurrentServer(CommonApplication.getAppContext())
                            ?.let { isCurrentServerSupportSpecScene(it) } == true
                    ) mPrefs.getInt("laboratory_scene2_switch", -1) else 0
                SceneManagerV2.scenePhoneFlag = mPrefs.getInt("phone_scene_switch", -1)
                shieldPhoneRecommendSwitch = mPrefs.getInt("shield_phone_recommend_switch", -1)
            }
            MiJiaLog.writeLogOnAll(
                LogType.SCENE,
                "grey check",
                "sceneCheckUserGray from cache ${SceneManagerV2.scenConvertFlag}   ${SceneManagerV2.specSceneEntranceFlag}   ${SceneManagerV2.scenePhoneFlag}   $shieldPhoneRecommendSwitch"
            )
        }

        fun getWallpaperPath(bgColor: String): String {
            val wallPaperConfig = WallPaperConfig.getInstance()
            return wallPaperConfig.getSceneWallPaperNamesByKey(bgColor)
                ?: wallPaperConfig.getSceneWallPaperNamesByKey("#EBF0F8")

        }

        fun getWallpaperBitmap(context: Context, wallpaperName: String): BitmapDrawable {

            val wallpaperPath = HomeManager.RoomIconManager.getWallPaperPath(wallpaperName)
            val bitmapFromPath = WallPaperUtils.getBitmapFromPath(wallpaperPath)

            return BitmapDrawable(context.resources, bitmapFromPath)
        }

//        fun getNameOfPhoneCondition(typeId: String?): String {
//            typeId ?: return ""
//            return when (typeId) {
//                "key_bluetooth_condition_item" -> "蓝牙"
//                "key_wlan_condition_item" -> "WLAN"
//                "key_location_condition_item" -> "定位"
//                "key_headset_condition_item" -> "有线耳机"
//                "key_hotspot_condition_item" -> "个人热点"
//                "key_start_activity_condition_item" -> "启动应用"
//                "key_leave_activity_condition_item" -> "离开应用"
//                "key_bluetooth_connect_device_condition_item" -> "连接蓝牙设备"
//                "key_bluetooth_disconnect_device_condition_item" -> "断开蓝牙设备"
//                "key_wlan_connect_specified_condition_item" -> "连接WLAN"
//                "key_wlan_disconnect_specified_condition_item" -> "断开WLAN"
//                "key_absorbed_condition_item" -> "专注模式"
//                "key_mute_condition_item" -> "静音/勿扰"
//                "key_charge_condition_item" -> "充电模式"
//                "key_battery_condition_item" -> "电池使用"
//                "key_lock_screen_condition_item" -> "屏幕锁定"
//                "key_incall_condition_item" -> "通话"
//                else -> ""
//            }
//        }

        //        fun notifySecurityCenterConditionDeleted(context: Context, triggers: List<Trigger>): Int {
//            val phoneTriggers = triggers.filter { TextUtils.equals(it.src, "phone") }
//            MiJiaLog.writeLogOnGrey(
//                LogType.SCENE,
//                "PhoneScene",
//                "delete ${phoneTriggers.size} phone condition"
//            )
//            if (phoneTriggers.isEmpty()) return 1
//            return try {
//                notifySecurityCenterDeleteCondition(
//                    context,
//                    phoneTriggers.map { it.key.split(".")[2] })
//            } catch (e: Exception) {
//                e.printStackTrace()
//                -1
//            }
//        }
        fun syncPhoneScenes2SafeCenter(context: Context) {
            if (!CoreApi.getInstance().isInternationalServer
                && isLogin()
                && SafeCenterHelper.isSupportPhoneRelated(context)
            ) {
                MiJiaLog.writeLogOnAll(
                    LogType.SCENE,
                    "ScenePhone",
                    "sync phone related scene to security center"
                )
                val phoneId = IdentifierManager.getOAID(context)
                val isOwnerHome = HomeManager.getInstance().currentHome?.isOwner ?: false
                val homeId = HomeManager.getInstance().currentHome?.id
                if (!isOwnerHome) {
                    MiJiaLog.writeLogOnAll(
                        LogType.SCENE,
                        "ScenePhone",
                        "sync scene fail : $homeId is not owner home"
                    )
                    return
                }
                val homeOwnerUid = HomeManager.getInstance().currentHome.ownerUid
                val successRun: ((result: JSONObject?) -> Unit) = { result ->
                    MiJiaLog.writeLogOnAll(LogType.SCENE, "ScenePhone", "sync scene sucess ")
                    if (result?.has("scene_info_list") == true) {
                        val sceneInfoList = result.optJSONArray("scene_info_list")
                        MiJiaLog.writeLogOnAll(
                            LogType.SCENE,
                            "ScenePhone",
                            "sceneInfo length: ${sceneInfoList?.length()} "
                        )
                        val specScenes = SpecScene.parseList(sceneInfoList, false)
                        MiJiaLog.writeLogOnGrey(
                            LogType.SCENE,
                            "ScenePhone",
                            "sync from main"
                        )
                        val uid = CoreApi.getInstance().miEncryptedUserId
                        val queryTasks =
                            SafeCenterHelper.queryTaskAllTask(context, uid)
                        val queryNeedDelete =
                            SafeCenterHelper.queryTaskAllTask(context, "0")
                        if (queryNeedDelete.isNotEmpty()) {
                            SafeCenterHelper.notifySecurityCenterDeleteScene(
                                context,
                                "0", queryNeedDelete.keys.map { it }
                            )
                        }

                        if (queryTasks.size != specScenes.size || specScenes.any {
                                !queryTasks.containsKey(it.id)
                            }) {
                            val syncre = syncAllTask(
                                context,
                                CoreApi.getInstance().miEncryptedUserId,
                                specScenes
                            )
                            MiJiaLog.writeLogOnAll(
                                LogType.SCENE,
                                "ScenePhone",
                                "has deleted scene $syncre"
                            )
                            val qSet = hashSetOf<String>().apply { addAll(queryTasks.keys) }
                            val invalidScenes = specScenes.filter { !qSet.contains(it.id) }
                            invalidScenes.forEach {
                                //最好能提示用户
                                MiJiaLog.writeLogOnAll(
                                    LogType.SCENE,
                                    "ScenePhone",
                                    "${it.id}=${it.name} invalid"
                                )
                            }
                            val sSet = specScenes?.map { it.id } ?: hashSetOf<String>()
                            val needDeletedScenes =
                                queryTasks.keys.filter { !sSet.contains(it) }
                            if (needDeletedScenes.isNotEmpty()) {
                                SafeCenterHelper.notifySecurityCenterDeleteScene(
                                    context,
                                    uid, needDeletedScenes
                                )
                            }
                        }
                        specScenes.forEach {
                            //所有创建的数据，保存的时候已经同步过，这里只需要同步开关状态即可。
                            if (!TextUtils.equals("${it.id}_${it.enable}", queryTasks[it.id])) {
                                SafeCenterHelper.notifySecurityCenterEnableChange(
                                    context,
                                    uid,
                                    it.id,
                                    it.enable
                                )
                            }
                        }
                    }
                }
                BaseSceneApi.getPhoneSceneList(
                    phoneId,
                    homeId,
                    homeOwnerUid.toString(),
                    object : AsyncCallback<JSONObject, Error>() {
                        override fun onSuccess(result: JSONObject?) {
                            successRun.invoke(result)
                        }

                        override fun onFailure(error: Error?) {
                            MiJiaLog.writeLogOnAll(
                                LogType.SCENE,
                                "ScenePhone",
                                "sync scene fail : getPhoneSceneList fail"
                            )
                        }

                    })
            }
        }

        /**
         * 在编辑页面调用的。入口考虑是否是当前手机,编辑页无需考虑
         */
        fun notifySecurityCenter(context: Context, ss: SpecScene, origin: SpecScene?) {
            if (origin == null) {//创建
                applyCreateOrUpdate(context, ss)
            } else {
                val originPhoneTriggers =
                    origin.triggers?.filter { TextUtils.equals(it.src, "phone") }
                val phoneTriggers = ss.triggers?.filter { TextUtils.equals(it.src, "phone") }
                val originHasPhoneTrigger = !originPhoneTriggers.isNullOrEmpty()
                val curHasPhoneTrigger = !phoneTriggers.isNullOrEmpty()
                if (originHasPhoneTrigger && !curHasPhoneTrigger) {
                    //原来有，现在没有
                    notifySecurityCenterDeleteScene(
                        context,
                        arrayListOf<SpecScene>().apply { add(origin) })
                } else if (curHasPhoneTrigger) {
                    //原来没有，现在有
                    //原来有，现在也有。直接sync？
                    applyCreateOrUpdate(context, ss)
                }
            }

        }

        fun applyCreateOrUpdate(context: Context, ss: SpecScene): Int {

            val phoneConditions = ss.triggers.filter { t ->
                TextUtils.equals(t.src, "phone")
            }.mapNotNull { t ->
//                val split = t.key.split(".")
//                if (split.size != 3) null

                val syncJson = (t.extra as? PhoneExtra)?.syncJson
                if (TextUtils.isEmpty(syncJson)) null
                Condition(
                    JSONObject(syncJson).optString("instanceId"),
                    (t.extra as? PhoneExtra)?.typeId ?: "",
                    syncJson!!
                )
            }
            MiJiaLog.writeLogOnAll(
                LogType.SCENE,
                "ScenePhone",
                "applyCreateOrUpdate after save ${ss.id}  ${ss.enable}----------------"
            )
            return SafeCenterHelper.applyCreateOrUpdate(
                context,
                CoreApi.getInstance().miEncryptedUserId,
                phoneConditions,
                ss.id,
                ss.enable
            )
        }

        //通知安全中心要删除某个任务(场景)， 安全中心会将该场景id关联的全部条件删除掉
        fun notifySecurityCenterDeleteScene(context: Context, ss: List<SpecScene>): Int {
            val curOaId = IdentifierManager.getOAID(context)
            if (TextUtils.isEmpty(curOaId)) {
                return -1
            }
            val phoneScenes =
                ss.filter {
                    TextUtils.equals(
                        curOaId,
                        it.extra?.optString("phone_id")
                    )
                }.map { it.id }
            phoneScenes.forEach {
                MiJiaLog.writeLogOnAll(
                    LogType.SCENE,
                    "ScenePhone",
                    " delete  item ${phoneScenes.size}  $it----------------"
                )
            }

            return SafeCenterHelper.notifySecurityCenterDeleteScene(
                context,
                CoreApi.getInstance().miEncryptedUserId,
                phoneScenes
            )
        }

        /**
         * 将当前应用内所有的自动任务同步给安全中心, 大量数据时请分批同步
         */
        fun syncAllTask(context: Context, userId: String, scenes: List<SpecScene>?): Int? {
            scenes ?: return 1

            if (TextUtils.isEmpty(userId) || TextUtils.equals(userId, "0")) {
                MiJiaLog.writeLogOnAll(
                    LogType.SCENE,
                    "ScenePhone",
                    "sync fail caused by userid = 0"
                )
                return -1
            }
            //准备数据，请业务方按实际情况初始化
            val taskList = mutableListOf<Task>()
            val curOaId = IdentifierManager.getOAID(context)
            MiJiaLog.writeLogOnAll(
                LogType.SCENE,
                "ScenePhone",
                " start sync item----------------"
            )
            scenes.forEach { scene ->
                val conditionList = mutableListOf<Condition>()
                scene.triggers?.forEach triggerEach@{ trigger ->
                    if (!TextUtils.equals("phone", trigger.src)) return@triggerEach
                    val phoneExtra = trigger.extra as? PhoneExtra
                    if (!TextUtils.equals(phoneExtra?.did, curOaId)) return@triggerEach

                    val instanceId = phoneExtra?.syncJson?.let { syncJsonStr ->
                        try {
                            val syncJson = JSONObject(syncJsonStr)
                            syncJson.optString("instanceId")
                        } catch (e: Exception) {
                            null
                        }

                    }
                    val typeId = phoneExtra?.typeId
                    if (TextUtils.isEmpty(instanceId) || TextUtils.isEmpty(typeId)) return@triggerEach

//                    val split = trigger.key.split(".")
//                    val typeId = split[1]
//                    val instanceId = split[2]
                    val syncJson = (trigger.extra as? PhoneExtra)?.syncJson ?: return@triggerEach
                    conditionList.add(
                        Condition(
                            instanceId!!, typeId!! /*条件类型id*/,
                            syncJson /*对应创建/修改条件时返回的argument_condition_sync_json*/
                        )
                    )
                }
                val task = Task(
                    ConstantType.SAFE_CENTER_CHANNEL,
                    userId,
                    scene.id,
                    if (scene.enable) 1 else 0/*开关状态*/,
                    1/*条件触发规则*/,
                    conditionList
                )
                taskList.add(task)
                MiJiaLog.writeLogOnAll(
                    LogType.SCENE,
                    "ScenePhone",
                    "sync item ${scene.id}   ${scene.enable}"
                )
            }
            MiJiaLog.writeLogOnAll(
                LogType.SCENE,
                "ScenePhone",
                " end sync item ${taskList.size}----------------"
            )
            if (taskList.isEmpty()) return 1
            return SafeCenterHelper.syncAllTask(context, userId, taskList)
        }

        fun getDisabledTypes(triggers: List<Trigger>?, curTrigger: Trigger?): List<String> {
            //筛选出当前以外的，所有手机相关trigger
            return triggers?.filter {
                it.extra is PhoneExtra && !TextUtils.equals(
                    it.key,
                    curTrigger?.key
                )
            }?.map { (it.extra as PhoneExtra).typeId } ?: mutableListOf()
        }

        fun getValueTypeBy(value: Any?): Int {
            value ?: return 5
            if (value is Boolean) {
                return 4
            }
            if (value is Int) {
                return 1
            }
            if (value is Float) {
                return 2
            }
            if (value is JSONObject) {
                val value1 = value as JSONObject?
                if (value1 != null) {
                    if (value1.has("min") || value1.has("max") || value1.has("equal")) {
                        return 3
                    }
                }
                return 6
            }
            return 5
        }

        fun openRecommendSceneDetail(
            context: Context,
            did: String? = null,
            tpl: RecommendTpl? = null,
            scene: SpecScene,
            editFrom: Int? = 0
        ) {
            if (tpl?.tempId == ConstantType.TPL_ID_FIND_PET) {
                CameraRouterFactory.getCameraManagerApi().startPetFoundActivity(context)
                return
            }
            val params = JSONObject()
            try {
                params.put("rectemplateid", scene.recommId)
                params.put("template_type", tpl?.templateType)
                params.put("iconurl", tpl?.logoUrl ?: "")
                params.put("edit_from", editFrom)
                if (!TextUtils.isEmpty(scene.roomId) && !TextUtils.equals(
                        scene.roomId, "0"
                    )
                ) {
                    params.put("room_id", scene.roomId)
                    val room = HomeManager.getInstance().getRoomById(
                        scene.roomId
                    )
                    if (room != null && !TextUtils.isEmpty(room.name)) {
                        params.put("room_name", room.name)
                    }
                }
                if (!TextUtils.isEmpty(did)) {
                    params.put("real_did", did)
                }
                params.put("scene_id", scene.id)
                if (tpl?.TCASingleDevice ?: isTCASingleDevice(scene.recommId)) {
                    params.put("trigger_single_device", true)
                    val subTemplates = tpl?.subTemplateIds ?: getSubtemplateIds(scene.recommId)
                    if (subTemplates != null) {
                        params.put("include", subTemplates)
                    }
                }
                SceneRouterFactory.getSceneBridge().openRecommendSceneDetail(context, params)

            } catch (e: Exception) {
                e.printStackTrace()
            }
        }

        fun getStateStr(
            tpl: RecommendTpl,
            category: RecommendTplCategory? = null,
            scenes: List<SpecScene>?
        ): Pair<CharSequence, Int> {
            //没有创建的场景或者创建的场景全部停用 展示
            return if (tpl.templateType == 3 || tpl.templateType == 4){
                if(tpl.canOpen == true ){
                    //TODO：by jiangmeidan 如果是手机设备，需要符合判断，当前手机是否支持该能力
                    Pair(
                        CommonApplication.getAppContext()
                            .getString( R.string.smart_tab_explore_template_card_content_can_open_type3),
                        CommonApplication.getAppContext().resources.getColor(R.color.mj_color_green_normal)
                    )
                }else{
                    Pair(
                        "",
                        CommonApplication.getAppContext().resources.getColor(R.color.mj_color_black_40_transparent)
                    )
                }

            }else if (scenes?.isNotEmpty() == true && scenes.find { it.enable } != null)
                Pair(
                    CommonApplication.getAppContext().getString(
//                        if ((category?.categoryId
//                                ?: 0) == ConstantType.CAT_ID_LATEST
//                        ) R.string.smart_tab_explore_template_card_app_content_is_opened else R.string.go_edit
                        R.string.go_edit//MIIO-78287
                    ),
                    CommonApplication.getAppContext().resources.getColor(
//                        if ((category?.categoryId
//                                ?: 0) == ConstantType.CAT_ID_LATEST
//                        ) R.color.mj_color_green_normal else R.color.mj_color_black_40_transparent
                        R.color.mj_color_black_50_transparent//MIIO-78287
                    )
                )
            else if (tpl.canOpen == true && !tpl.trigger.options.any { op ->
                    TextUtils.equals(
                        op.triggeType,
                        "phone"
                    )
                })
                Pair(
                    CommonApplication.getAppContext()
                        .getString( R.string.smart_tab_explore_template_card_content_can_open),
                    CommonApplication.getAppContext().resources.getColor(R.color.mj_color_function_brandtext_normal)
                )
            else Pair(
                "",
                CommonApplication.getAppContext().resources.getColor(R.color.mj_color_black_40_transparent)
            )
        }

        fun getOpenQuantityState(tpl: RecommendTpl): String {
            if (tpl.tempId == ConstantType.TPL_ID_FIND_PET) {
                return if (!TextUtils.isEmpty(tpl.provider)) tpl.provider!!
                else ""
            }
            return if (!TextUtils.isEmpty(tpl.provider)) {
                tpl.provider!!
            } else if (System.currentTimeMillis() - ((tpl.createTime
                    ?: 0) * 1000L) < 1000L * 60 * 60 * 24 * 30
            ) {
                CommonApplication.getAppContext().resources.getString(R.string.smart_tab_explore_template_content_opened_latest_30d)
            } else if ((tpl.openQuantity ?: 0) < 10000) {
                CommonApplication.getAppContext().resources.getString(R.string.smart_tab_explore_template_content_opened_less_than_1m)
            } else {
                val realNumMsg = if (Locale.CHINA.toString()
                        .equals(Locale.getDefault().toString(), ignoreCase = true)
                    || Locale.CHINESE.toString()
                        .equals(Locale.getDefault().language.toString(), ignoreCase = true)
                ) {
                    (tpl.openQuantity ?: 0) / 10000
                } else {
                    (tpl.openQuantity ?: 0) / 1000
                }
                val realMsgStr = if (Locale.CHINA.toString()
                        .equals(Locale.getDefault().toString(), ignoreCase = true)
                    || Locale.CHINESE.toString()
                        .equals(Locale.getDefault().language.toString(), ignoreCase = true)
                ) {
                    "${realNumMsg}万"
                } else {
                    "${realNumMsg}K"
                }
                CommonApplication.getAppContext().resources.getQuantityString(
                    R.plurals.smart_tab_explore_template_content_opened_count,
                    realNumMsg,
                    realMsgStr
                )
            }
        }

        private fun isTCASingleDevice(tplId: Long): Boolean {
            return tplId == ConstantType.TPL_ID_MOTION
        }

        private fun getSubtemplateIds(tplId: Long): JSONArray? {
            return when (tplId) {
                ConstantType.TPL_ID_MOTION -> {
                    JSONArray().apply { put(ConstantType.TPL_ID_MOTION_SUB) }
                }

                ConstantType.TPL_ID_OPEN_NO_DISTURB -> {
                    JSONArray().apply { put(ConstantType.TPL_ID_CLOSE_NO_DISTURB) }
                }

                else -> null
            }
        }

        fun getPhoneTrigger(phoneTtriggers: List<Trigger>): List<Trigger> {
            return phoneTtriggers.filter { TextUtils.equals("phone", it.src) }
        }

        fun hasOtherPhoneTrigger(triggers: List<Trigger>, currentPhoneId: String?): List<Trigger> {
            return triggers.filter { trigger ->
                !TextUtils.equals((trigger.extra as? PhoneExtra)?.did, currentPhoneId)
            }
        }

        fun doAfterSaveRecScene(activity: FragmentActivity?, data: Intent) {
            activity?.also {
                LocalBroadcastManager.getInstance(it).sendBroadcast(
                    Intent(
                        SceneListViewModel.ACTION_REFRESH_LIST
                    )
                )
            }
            val jsonStr = data.getStringExtra("rec_scene_plugin")
            val jsonObject = try {
                JSONObject(jsonStr)
            } catch (e: Exception) {
                null
            }
            jsonObject?.also { resPluginData ->
                val editType = resPluginData.optString("type")
                if (TextUtils.equals(editType, "create")) {
                    val tsid = resPluginData.optString("sceneId")
                    RecommendSceneManager.getInstance().newSceneId = tsid
                }
            }
            if (TextUtils.equals(jsonObject?.optString("type"), "delete")) {
                return
            }
            activity?.also { a ->
                (a as CommonActivity).mHandler.postDelayed({
                    ToastUtil.show(
                        if (TextUtils.equals(
                                jsonObject?.optString("type"), "create"
                            )
                        ) R.string.smart_tab_explore_template_card_content_open_success else R.string.rec_scene_edit_success
                    )
                }, 100)
            }
        }

        fun getPhoneRelatedDefaultTrigger(
            did: String?,
            deviceName: String,
            triggerExtraSyncJson: String,
            k: String
        ): Trigger? {//华为手机 Exception in HostFunction: java.lang.NullPointerException: Parameter specified as non-null is null: method com.xiaomi.smarthome.specscene.util.SpecSceneHelper$Companion.getPhoneRelatedDefaultTrigger, parameter did, js engine: v8, stack:
            return when (k) {
                "key_charge_condition_item" -> Trigger().apply {
                    src = "phone"
                    key = "prop.7.2"
                    valueJson = 1
                    valueType = 1
                    extra = PhoneExtra(
                        CommonApplication.getAppContext()
                            .getString(R.string.summary_condition_in_charge),
                        triggerExtraSyncJson,
                        k,
                        did,
                        deviceName,
                        ConstantType.PHONE_MODEL_V2
                    )
                    name =
                        CommonApplication.getAppContext().getString(R.string.title_condition_charge)

                }

                "key_mute_condition_item" -> Trigger().apply {
                    name =
                        CommonApplication.getAppContext().getString(R.string.title_condition_mute)
                    src = "phone"
                    key = "prop.5.2"
                    valueJson = true//false
                    valueType = 4
                    extra = PhoneExtra(
                        CommonApplication.getAppContext()
                            .getString(R.string.task_summary_open_mute_mode),
                        triggerExtraSyncJson,
                        k,
                        did,
                        deviceName,
                        ConstantType.PHONE_MODEL_V2
                    )
                }

                else -> null
            }
        }

        fun setImg(imageView: SimpleDraweeView, strUrl: String?, size: Int = 40f.dp) {
            strUrl?.also {
                val imageLayoutParam = imageView.layoutParams
                imageLayoutParam.height = size
                imageLayoutParam.width = size
                imageView.layoutParams = imageLayoutParam
                if (TextUtils.equals(it, (imageView.tag as? String))) {
                    return@also
                }
                imageView.tag = it
                val hierarchy =
                    GenericDraweeHierarchyBuilder.newInstance(imageView.context.resources)
                        .setFadeDuration(200)
                        .setPlaceholderImage(imageView.context.resources.getDrawable(R.drawable.device_list_phone_no))
                        .setPlaceholderImageScaleType(ScalingUtils.ScaleType.CENTER_INSIDE)
                        .build()
                imageView.hierarchy = hierarchy
                val realUrl = Uri.parse(it)
                val mRequestBuilder = ImageRequestBuilder.newBuilderWithSource(realUrl)
                    .setResizeOptions(ResizeOptions(size, size))

                val controller: DraweeController = Fresco.newDraweeControllerBuilder()
                    .setOldController(imageView.controller)
                    .setImageRequest(mRequestBuilder.build())
                    .build()
                    .apply {
                        addControllerListener(object : BaseControllerListener<Any?>() {
                            override fun onFailure(id: String?, throwable: Throwable?) {
                                try {
                                    Fresco.getImagePipeline().evictFromMemoryCache(realUrl)
                                    Fresco.getImagePipeline().evictFromDiskCache(realUrl)
                                } catch (e: java.lang.Exception) {

                                }
                            }
                        })
                    }

                imageView.controller = controller
            }

        }

        private fun initGeofence() {
            isSupportGeofence = SharePrefsManager.getSettingBoolean(
                CommonApplication.getAppContext(),
                SharePrefsManager.SP_KEY_SCENE_HARDWARE,
                "is_support_geofence",
                false
            )
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                val api = MiJiaRouter.getService(
                    IMIUIGeoFenceManagerApi::class.java, IMIUIGeoFenceManagerApi.IMPL_NAME
                )
                    ?: return
                MIUIGeoFenceManager.getInstance()
                    .isSupportedAsync(object : AsyncCallback<Boolean, Error>() {
                        override fun onSuccess(result: Boolean) {
                            MiJiaLog.onlyLogcat(
                                "jiangmd236",
                                "SpecSceneHelper issupport ${result} "
                            )
                            isSupportGeofence = result
                            SharePrefsManager.setSettingBoolean(
                                CommonApplication.getAppContext(),
                                SharePrefsManager.SP_KEY_SCENE_HARDWARE,
                                "is_support_geofence",
                                result
                            )
                        }

                        override fun onFailure(error: Error) {}
                    })
            }
        }

        fun isRecVisibleInSceneList(tags: JSONObject?): Boolean {
            if (tags?.has("is_visible") == true) {
                return tags.optBoolean("is_visible", false)
            }
            return !ConstantType.hideTags.contains(tags?.optString("classification") ?: "")
        }

        fun isHmSceneInSceneList(tags: JSONObject?): Boolean {
            if (tags?.has("source") == true) {
                return tags.optString("source").equals("hyper_mind_v2")
            }
            return false
        }

        fun getCurrentHomeScene1(sce: SmartHomeScene, home: Home): SmartHomeScene? {
            if (home.permitLevel < Home.PERMIT_HOME_OWNER) return null
            if (sce.authedDids.size == 0) return sce
            val homeApi = MiJiaRouter.getService(
                IHomeManagerApi::class.java, IHomeManagerApi.KEY_HOME
            )

            val didHomeMap = homeApi.didHomeMap
            val homeIds: MutableSet<String> = HashSet() //统计跨了哪些家庭

            sce.authedDids.forEach { did ->
                didHomeMap[did]?.also { th -> homeIds.add(th.id) }
            }

            if (!homeIds.contains(home.id)) { //参与的设备都不是当前家庭的
                return null
            }
            return sce
        }

        private fun getRecommendSceneUiConfigV2(context: Context): Map<String, JSONObject> {
            if (recommendTplIconInfo.isNotEmpty()) return recommendTplIconInfo
            val stringBuilder = StringBuilder()
            val bf = BufferedReader(
                InputStreamReader(
                    context.assets.open("recommend_scene_ui_config_v2.json")
                )
            )
            var line: String?
            while (bf.readLine().also { line = it } != null) {
                stringBuilder.append(line)
            }
            val array = try {
                try {
                    JSONObject(stringBuilder.toString())
                } catch (e: Exception) {
                    null
                }?.optJSONArray("data")
            } catch (e: Exception) {
                null
            }
            recommendTplIconInfo.clear()
            for (i in 0 until (array?.length() ?: 0)) {
                try {
                    array?.optJSONObject(i)
                } catch (e: Exception) {
                    null
                }?.also { oneConfig ->
                    val tempId: String = oneConfig.optString("temp_id")
                    if (!TextUtils.isEmpty(tempId)) {
                        recommendTplIconInfo[tempId] = oneConfig
                    } else {
                        recommendTplIconInfo[oneConfig.optString("widget_icon")] = oneConfig
                    }
                }
            }
            return recommendTplIconInfo
        }

        fun getRecTplIconBy(context: Context?, tplId: String?): String {
            tplId ?: return ConstantType.USER_CLICK_ICON_URL
            context?.also { getRecommendSceneUiConfigV2(it) }
            return recommendTplIconInfo.getOrElse(tplId) { null }?.optString("widget_icon")
                ?: ConstantType.USER_CLICK_ICON_URL
        }

        fun getWidgetSceneSelections(
            context: Context,
            homeId: String = HomeManager.getInstance().currentHomeId,
            did: String? = null,
            listener: IScenceListener? = null
        ): AsyncHandle {
            val asyncHandle = AsyncHandle()
            val cancelableRef = CancelableRef(asyncHandle, listener)
            val home = HomeManager.getInstance().getHomeById(homeId)
            home?.also {
                val re: ArrayList<WidgetScene> = arrayListOf()
                val refConf = WeakReference(context)
                val con = refConf.get() ?: return asyncHandle
                val configMap = getRecommendSceneUiConfigV2(con)
                val config1 = configMap["1"] ?: configMap["rec_custom_user_click"]
                val scene2Fun = fun(result: JSONObject?) {
                    val array = result?.optJSONArray("scene_info_list")
                    for (i in 0 until (array?.length() ?: 0)) {
                        array?.optJSONObject(i)?.let { sceneJson -> SpecScene.parse(sceneJson) }
                            ?.also { scene ->
                                val tmpConfig =
                                    if (scene.recommId > 0) configMap[scene.recommId.toString()] else if (!TextUtils.isEmpty(
                                            scene.iconUrl
                                        ) && scene.iconUrl?.startsWith(
                                            "https://"
                                        ) == true
                                    ) configMap[scene.iconUrl] else config1
                                if (scene.isUserClickScene && scene.enable && tmpConfig != null) {
                                    re.add(WidgetScene().apply {

                                        id = scene.id
                                        name = scene.name
                                        sceneType = 2
                                        iconUrl = tmpConfig.optString("widget_icon")
//                                            darkIconUrl =
//                                                if (DarkModeCompat.isDarkMode(con)) iconUrl.replace(
//                                                    ".png",
//                                                    "Night.png"
//                                                ) else iconUrl
                                        bgUrl = tmpConfig.optString("widget_bg")
                                        bgColor =
                                            tmpConfig.optString("widget_icon_bg_color") ?: ""
                                        shadowUrl = tmpConfig.optString("widget_shadow") ?: ""
                                    })
                                }
                            }
                    }
                }
                RemoteApi.getTypedSceneList(
                    context,
                    it.id,
                    it.ownerUid,
                    did,
                    0,
                    object : AsyncCallback<JSONObject, Error>() {
                        override fun onSuccess(result: JSONObject?) {
                            scene2Fun.invoke(result)
                            val refConfInner = WeakReference(con)
                            RemoteSceneInnerApi.getInstance().getRemoteScenes(con,
                                PluginRecommendSceneManager.API_LEVEL,
                                30,
                                did,
                                "",
                                false,
                                null,
                                object : AsyncCallback<JSONObject, Error>() {
                                    override fun onSuccess(result: JSONObject?) {
                                        val conInner = refConfInner.get() ?: return
                                        for (i in 0 until 500) {//1.0手动控制最多能添加256个
                                            if (result?.has(i.toString()) == false) break
                                            result?.optJSONObject(i.toString())?.let { sceneJson ->
                                                SmartHomeScene.parseFromJsonNew(
                                                    sceneJson,
                                                    home,
                                                    30
                                                )
                                            }?.also { scene ->
                                                if (!TextUtils.isEmpty(scene.id) && !TextUtils.equals(
                                                        "0",
                                                        scene.id
                                                    )
                                                ) {
                                                    re.add(WidgetScene().apply {
                                                        id = scene.id
                                                        name = scene.name
                                                        sceneType = 1
                                                        iconUrl = ConstantType.USER_CLICK_ICON_URL
//                                                darkIconUrl =
//                                                    if (DarkModeCompat.isDarkMode(con)) iconUrl.replace(
//                                                        ".png",
//                                                        "Night.png"
//                                                    ) else iconUrl
                                                        bgColor = "#5C28FF"
                                                        shadowUrl =
                                                            "https://cnbj1.fds.api.xiaomi.com/scenetemplate/scene_picture/Widget_PurpleShadow.png"
                                                    })
                                                }
                                            }
                                        }
                                        val refConfInner2 = WeakReference(conInner)
                                        RemoteApi.getTypedSceneList(
                                            context,
                                            it.id,
                                            it.ownerUid,
                                            did,
                                            1,
                                            object : AsyncCallback<JSONObject, Error>() {
                                                override fun onSuccess(result: JSONObject?) {
                                                    refConfInner2.get() ?: return
                                                    scene2Fun.invoke(result)
                                                    cancelableRef.get()?.onRefreshScenceSuccess(
                                                        SceneManagerV2.REFRESH_LAST,
                                                        re
                                                    )
                                                }

                                                override fun onFailure(error: Error?) {
                                                    cancelableRef.get()?.onRefreshScenceSuccess(
                                                        SceneManagerV2.REFRESH_LAST,
                                                        re
                                                    )
                                                }
                                            })
                                    }

                                    override fun onFailure(error: Error?) {
                                        cancelableRef.get()?.onRefreshScenceSuccess(
                                            SceneManagerV2.REFRESH_LAST,
                                            re
                                        )
                                    }
                                })
                        }

                        override fun onFailure(error: Error?) {
                            cancelableRef.get()?.onRefreshScenceFailed(SceneManagerV2.REFRESH_LAST)
                        }
                    })
            } ?: run {
                cancelableRef.get()?.onRefreshScenceFailed(SceneManagerV2.REFRESH_LAST)
            }

            return asyncHandle
        }

        fun getSpecItem(
            instance: SpecDevice,
            prefix: String,
            sTypedName: String,
            xTypedName: String,
            filter: ((specItems: List<Spec.SpecItem>) -> List<Spec.SpecItem>)? = null
        ) = SpecUtils.getSpecItemMap(instance) { spec ->
            "miot-spec-v2" == spec.schema
        }["${prefix}:${sTypedName}:${xTypedName}"]?.let { specItems ->
//            specItems[0]
            filter?.invoke(specItems)
        }

        fun getKeyWordFromSpecItemDesc(tmpKey: String) =
            if (tmpKey.lowercase().contains("left")) "left"
            else if (tmpKey.lowercase().contains("right")) "right"
            else if (tmpKey.lowercase().contains("middle")) "middle"
            else if (tmpKey.lowercase().contains("first")) "first"
            else if (tmpKey.lowercase().contains("second")) "second"
            else if (tmpKey.lowercase().contains("third")) "third"
            else if (tmpKey.lowercase().contains("fourth")) "fourth"
            else if (tmpKey.lowercase().contains("fifth")) "fifth"
            else if (tmpKey.lowercase().contains("sixth")) "sixth"
            else if (tmpKey.lowercase().contains("top")) "top"
            else if (tmpKey.lowercase().contains("bottom")) "bottom"
            else "other"

        fun isSoloPhone(model: String?): Boolean {
            model ?: return false
            return SmartHomeDeviceManager.isSoloPhone(model)
        }
//        fun isPhoneScene(scene:SortSceneData,checktrigger:Boolean = true,checkCondition: Boolean = true,checkAction:Boolean = true):Boolean{
//            return isV2PhoneScene(scene) || isV3PhoneScene(scene,checktrigger,checkCondition,checkAction)
//        }

        fun isV3PhoneScene(
            scene: SortSceneData,
            checktrigger: Boolean = true,
            checkCondition: Boolean = true,
            checkAction: Boolean = true
        ): Boolean {
            if (scene is SimpleScene && scene.tags?.optString(ConstantType.TAG_IOT_PHONE) == "true") return true
            return if (scene is SpecScene) {
                (checktrigger && scene.triggers.any { (it.extra as? DeviceExtra)?.isSoloPhoneExtra == true }) ||
                        (checkCondition && scene.conditions.any { (it.extra as? DeviceExtra)?.isSoloPhoneExtra == true }) ||
                        (checkAction && scene.actions.any { (it.payload as? RpcPayload)?.isSoloPhonePayload == true })
            } else false
        }

        fun isWearScene(
            scene: SortSceneData,
            checktrigger: Boolean = true,
            checkCondition: Boolean = true,
            checkAction: Boolean = true
        ): Boolean {
            if (scene is SimpleScene && scene.tags?.optString(ConstantType.TAG_IOT_WEAR) == "true") return true
            return if (scene is SpecScene) {
                (checktrigger && scene.triggers.any { SmartHomeDeviceManager.isWatch((it.extra as? DeviceExtra)?.model) }) ||
                        (checkCondition && scene.conditions.any {
                            SmartHomeDeviceManager.isWatch(
                                (it.extra as? DeviceExtra)?.model
                            )
                        }) ||
                        (checkAction && scene.actions.any { SmartHomeDeviceManager.isWatch((it.payload as? RpcPayload)?.model) })
            } else false
        }

        fun isV2PhoneScene(scene: SortSceneData): Boolean {
            if (scene is SimpleScene && scene.tags?.optString("classification") == "phone") return true
            return if (scene is SpecScene) {
                scene.triggers.any { it.extra is PhoneExtra }
            } else false
        }

        fun isMine(
            scene: SpecScene,
            checktrigger: Boolean = true,
            checkCondition: Boolean = true,
            checkAction: Boolean = true
        ): Boolean {
            val bindedPhone = getPhoneV3()?.did?:return false
            var phoneId =
                if (checktrigger) (scene.triggers.find { bindedPhone == (it.extra as? DeviceExtra)?.did }?.extra as? DeviceExtra)?.did else null
            phoneId = phoneId
                ?: if (checkCondition) (scene.conditions.find { bindedPhone == (it.extra as? DeviceExtra)?.did }?.extra as? DeviceExtra)?.did else null
            phoneId = phoneId
                ?: if (checkAction) (scene.actions.find { bindedPhone == (it.payload as? RpcPayload)?.did }?.payload as? RpcPayload)?.did else null
            phoneId ?: return false
            return bindedPhone == phoneId
        }

        fun getPhoneV3(): Device? {
            return SmartHomeDeviceManager.getInstance().findSoloDevices()
                .find { SmartHomeDeviceManager.isSoloPhone(it.model) && it.did.equals(SmartHomeDeviceHelper.getCurPhoneDid()) }
        }

        fun getPhoneDevice(type: String = "online"): Device = Device().apply {
            did = if (type == "online") ConstantType.MODEL_MPHONE else IdentifierManager.getOAID(
                ServiceApplication.getAppContext()
            )
            model = ConstantType.MODEL_MPHONE
            name = Settings.Global.getString(
                ServiceApplication.getAppContext().contentResolver,
                Settings.Global.DEVICE_NAME
            ) ?: ""
            icon = getResouceUriString(R.drawable.mijia_phone_scene)
        }

        fun toInhomePage(
            activity: BaseActivity,
            launcher: ActivityResultLauncher<Intent>? = null,
            onClickNagtive: (() -> Unit)? = null,
            onDismiss: (() -> Unit)? = null

        ) {
            val inHomeItem = findPersonItemByPersonId("2.1.1")
            val items: List<OptionClickViewContent.SubItem> = listOf(
                OptionClickViewContent.SubItem(inHomeItem?.name ?:activity.getString(R.string.at_home_nav_sec), id = 63257, iconUrl = inHomeItem?.icon ?: "")
            )
            val builder = MJOptionClickDialog.Builder(
                activity,
                items,
                color = com.xiaomi.smarthome.widget.R.color.mj_color_gray_dialog_2,
                true,
                activity.getString(R.string.only_valid_at_home_message_dia_message)
            )

            fun startPersonHomeSettingActivityWithExtras(title: String) {
                val intent = Intent(activity, PersonHomeSetting::class.java)
                intent.putExtra("title", inHomeItem?.name)
                intent.putExtra("person_id", inHomeItem?.id)
                intent.putExtra("isEdit", false)
                intent.putExtra("request_code", ACTION_SELECT_CONDITION)
                intent.putExtra("mode", "condition")
                if (launcher == null) {
                    activity.startActivityForResult(intent, ACTION_SELECT_CONDITION)
                } else {
                    launcher.launch(intent)
                }

            }
            val dialog = builder
                .setTitle(activity.getString(R.string.only_valid_at_home_title_dia_title))
                .onPositiveClick(activity.getString(R.string.smart_config_goto_setting)) {
                    startPersonHomeSettingActivityWithExtras(activity.getString((R.string.at_home_nav_sec)))
                }
                .onNegativeClick(activity.getString(R.string.jump_over)) {
                    onClickNagtive?.invoke()
                }
                .onDismiss {
                    onDismiss?.invoke()
                }
                .show()
            builder.getResult.onItemClick { itemView, idx, index ->
                startPersonHomeSettingActivityWithExtras(activity.getString((R.string.at_home_nav_sec)))
                dialog?.dismiss()
            }

        }

        fun isNonPOIPhoneSene(dModel: String?, nTrigger: Trigger?): Boolean {
            if (!SmartHomeDeviceManager.isSoloPhone(dModel)) {
                return false
            }
            if (nTrigger?.key == "prop.2.1.4" || nTrigger?.key == "prop.2.13.2") {
                return false
            }
            return true
        }

        fun hasSoloDeviceNotIn(tmpScene: SpecScene, useProductDefine :Boolean = false): Boolean {
            //useLocal true 不看真实状态，看产品定义的规则。false  看设备真实状态，改展示已删除就展示
            val hasSoloPhone = tmpScene.tags?.optBoolean(ConstantType.TAG_IOT_PHONE) == true
            val hasWearPhone = tmpScene.tags?.optBoolean(ConstantType.TAG_IOT_WEAR) == true
            val curUid = CoreApi.getInstance().miId
            val soloPhonesInScene = hashSetOf<String>()
            val soloPhonesSceneAuth = hashSetOf<String>()
            val soloWearInScene = hashSetOf<String>()
            val soloWearSceneAuth = hashSetOf<String>()
            if (hasSoloPhone || hasWearPhone) {
                fun findSoloDevicesInScene(type: String?, tDid: String, uid: String? = null) {
                    if (type == "phone") {
                        soloPhonesInScene.add(tDid)
                        uid?.also {
                            soloPhonesSceneAuth.add(it)
                        }
                    } else if (type == "miwear") {
                        soloWearInScene.add(tDid)
                        uid?.also {
                            soloWearSceneAuth.add(it)
                        }

                    }
                }
                tmpScene.triggers.forEach {
                    val type = (it.extra as? DeviceExtra)?.deviceType
                    (it.extra as? DeviceExtra)?.did?.also { tDid->
                        findSoloDevicesInScene(type,tDid,(it.extra as? DeviceExtra)?.owneruid)
                    }
                }
                tmpScene.conditions.forEach {
                    val type = (it.extra as? DeviceExtra)?.deviceType
                    (it.extra as? DeviceExtra)?.did?.also { tDid->
                        findSoloDevicesInScene(type,tDid,(it.extra as? DeviceExtra)?.owneruid)
                    }
                }
                tmpScene.actions.forEach {
                    val type = (it.payload as? RpcPayload)?.deviceType
                    (it.payload as? RpcPayload)?.did?.also { tDid->
                        findSoloDevicesInScene(type,tDid,(it.payload as? RpcPayload)?.ownerUid)
                    }
                }
                var illegal = false
                if(soloPhonesInScene.isNotEmpty()){
                    val curPhoneDId = if (useProductDefine) {//固件did
                        SmartHomeDeviceHelper.getCurPhoneDid()
                    } else {
                        SmartHomeDeviceManager.getInstance().findSoloDevices()
                            .find { it.isSoloDevice && SmartHomeDeviceManager.isSoloPhone(it.model) }?.did
                    }
                    illegal = soloPhonesInScene.any {
                        curPhoneDId != it && !SmartHomeDeviceManager.getInstance().findSoloDevices()
                            .any { solo -> solo.did == it }
                    } || soloPhonesSceneAuth.any { curUid != it }
                }
                if(!illegal && soloWearInScene.isNotEmpty()){
                    illegal = if (useProductDefine){
                        soloWearSceneAuth.any { curUid != it }
                    }else{
                        soloWearInScene.any {
                            SmartHomeDeviceManager.getInstance().findSoloDevices()
                                .any { solo -> solo.did != it }
                        }
                    }
                }
                return illegal
            }
            return false
        }

        fun canRename(ss: SimpleScene?): Boolean {
            if (ss?.isEditable == false) return false
            if (isHmSceneInSceneList(ss?.tags)) return false
            val detail = SceneManagerV2.getInstance().getSceneById(ss?.id) ?: return false
            if (detail.recommId > 0) return false
            if (isV3PhoneScene(detail,
                    checktrigger = true,
                    checkCondition = true,
                    checkAction = true
                )) {
                if (detail is SpecScene && hasSoloDeviceNotIn(detail,false)) {
                    return false
                }
            }
            if(isWearScene(detail, checktrigger = true, checkCondition = true, checkAction = false)
                && HomeManager.getInstance().currentHome?.isOwner == false
                && !getAdminWearSupport()){
                return false
            }
            val copyScene =  SceneManagerV2.getInstance().getSceneById(ss?.id) as? SpecScene
            val personError =
                (copyScene?.let { hasPerceptionOwnerScene(it) } == true && !HomeManager.getInstance().currentHome.isOwner)
                        ||
                        copyScene?.let { isBetaHmError(it.triggers, it.conditions) } == true
            val necessaryError = copyScene?.let { isNecessaryError(it) }
            val ownerError =  copyScene?.tags?.optBoolean(ConstantType.TAG_IOT_PERCEPTION) == true && !HomeManager.getInstance().currentHome.isOwner

            val deviceError = copyScene?.let { hasSoloDeviceNotIn(it,true) }
            //车车
            val carError = copyScene?.let { carError(it) }
            //地理围栏
            val locationError = copyScene?.let {locationError(it)}
            return !(carError == true || locationError == true || deviceError == true || personError || ownerError || necessaryError == true)
        }

        fun isPerceptionScene(
            scene: SortSceneData,
            checktrigger: Boolean = true,
            checkCondition: Boolean = true,
            checkAction: Boolean = true,
        ): Boolean {
            if (scene is SimpleScene && scene.tags?.optString(ConstantType.TAG_IOT_PERCEPTION) == "true") return true
            return if (scene is SpecScene) {
                (checktrigger && scene.triggers.any { (it.extra as? DeviceExtra)?.isPerceptionExtra==true}) ||
                        (checkCondition && scene.conditions.any {
                            (it.extra as? DeviceExtra)?.isPerceptionExtra==true
                        })
            } else false
        }

        fun getValueByParams(params: JSONArray?): String {
            // 检查 params 是否为空且长度是否大于 0，返回 "value" 或空字符串
            return params?.takeIf { it.length() > 0 }
                ?.optJSONObject(0)
                ?.optString("value") ?: ""
        }

        fun hasPerceptionOwnerScene(tmpScene: SpecScene): Boolean {
            //判断我回家 我离家 我在家 针对我 取uid
            val hasPerceptionScene = tmpScene.tags?.optBoolean(ConstantType.TAG_IOT_PERCEPTION) == true
            if (hasPerceptionScene) {
                val isOwner = HomeManager.getInstance().currentHome?.isOwner
                if (isOwner == false) return true //管理员走禁止编辑
                tmpScene.triggers.any {
                    val valueType =it.valueType
                    val deviceExtra = (it.extra as? DeviceExtra)
                    if (deviceExtra != null) {
                        return PersonalStatusManager.getPersonSubtitle(it.tcaId.toString(),deviceExtra,valueType,it.valueJson).third
                    }
                    else return false
                }
                tmpScene.conditions.any {
                    val valueType =it.valueType
                    val deviceExtra = (it.extra as? DeviceExtra)
                    if (deviceExtra != null) {
                        return PersonalStatusManager.getPersonSubtitle(it.tcaId.toString(),deviceExtra,valueType,it.valueJson).third
                    }
                    else return false
                }
                return false
            }
            else{
                return false
            }
        }
        fun getDeviceStatusText(): Triple<CharSequence, Boolean, Boolean> {
            val status =  XiaoaiDeviceManager.getXiaoDeviceStatus()
            return status
        }

        fun notHasQuickActionDevice(): Boolean {
            return QuickActionManager.notHasDevice()
        }
        //在家下面有可以配置的
        fun isSupportInhome(): Boolean {
            val personItem = findPersonItemByPersonId("2.1.1") ?: return false
            return (personItem.perceptivescenes
                ?.any {
                    oneItemShow(it)
                } == true)
                    ||
                    (personItem.list.isNotEmpty() && getPhoneV3() != null && isSupportGeofence)
        }

        fun hasPerceptionSceneDetails(type: String): Boolean {
            return PersonalStatusManager.hasTriggerConditionScene(type)
        }
        fun getPersomImg(tcaId:Int) : String? {
            //这里需要根据persongroup获取
            return findPersonItemImgBySceneId(tcaId.toString())
        }
        fun getAccuracyBySceneId(sceneId:String): Int? {
           return PersonalStatusManager.getAccuracyBySceneId(sceneId)
        }
        fun getPerceptionPrivacyData(sceneId: Int): ScenePrivacy? {
            return PersonalStatusManager.getPerceptionPrivacyData(sceneId)
        }
        fun isPhoneSupport() :Boolean{
            val uid = CoreApi.getInstance().miId
            val isSameUidWithSystem = SystemAccountCompare.Companion.isSameAccountWithSystem(CommonApplication.getAppContext(),uid)
            val isV3 = (getPhoneV3()?.did != null)
            return isV3 && isSameUidWithSystem
        }
        fun isNecessaryError(tmpScene: SpecScene): Boolean {
            val isError = tmpScene.triggers.any {
                (it.extra as? DeviceExtra)?.deviceType == ConstantType.PERCEPTION && getPerceptionPrivacyData(it.tcaId)?.phone?.necessary == 1 }
             || tmpScene.conditions.any {
                (it.extra as? DeviceExtra)?.deviceType == ConstantType.PERCEPTION && getPerceptionPrivacyData(it.tcaId)?.phone?.necessary == 1 }
            return isError && !isPhoneSupport()
        }
        fun isBetaHmError(triggers: ArrayList<SpecScene.Trigger>, conditions: ArrayList<SpecScene.Condition>): Boolean {
            return triggers.any {
                (it.extra as? DeviceExtra)?.deviceType == ConstantType.PERCEPTION && (if (SpecSceneHelper.onePersonItemError(
                        it, "trigger"
                    ).second
                ) true else {
                    SpecSceneHelper.onePersonItemError(it, "trigger").first.isNotEmpty()
                })
            } || conditions.any {
                (it.extra as? DeviceExtra)?.deviceType == ConstantType.PERCEPTION && (if (SpecSceneHelper.onePersonItemError(
                        it, "condition"
                    ).second
                ) true else {
                    SpecSceneHelper.onePersonItemError(it, "condition").first.isNotEmpty()
                })
            }
        }

        fun onePersonItemError(_content: SpecSceneItem, type: String): Pair<CharSequence, Boolean> {
            if (type == "trigger") {
                val item = _content as? Trigger
                if (item != null) {
                    val extra = item.extra as? DeviceExtra
                    val status = extra?.let {
                        PersonalStatusManager.getPersonSubtitle(
                            _content.tcaId.toString(),
                            it,
                            item.valueType,
                            item.valueJson
                        )
                    }
                    val subTitle = status?.first ?: ""
                    val isGrey = status?.second ?: true

                    return Pair(subTitle, isGrey)
                }
            } else {
                val item = _content as? SpecScene.Condition
                if (item != null) {
                    val extra = item.extra as? DeviceExtra
                    val status = extra?.let {
                        PersonalStatusManager.getPersonSubtitle(
                            _content.tcaId.toString(),
                            it,
                            item.valueType,
                            item.valueJson
                        )
                    }
                    val subTitle = status?.first ?: ""
                    val isGrey = status?.second ?: true

                    return Pair(subTitle, isGrey)
                }
            }
            return Pair("", false)
        }

        fun carError(tmpScene: SpecScene): Boolean {
            val ownerUid = CoreApi.getInstance().miId
            val sceneId = tmpScene
            tmpScene.triggers.forEach { trigger ->
                val deviceExtra = trigger.extra as? DeviceExtra
                if (deviceExtra != null) {
                    if (findTip(
                            deviceExtra.deviceType,
                            deviceExtra.did,
                            deviceExtra.deviceName,
                            deviceExtra.owneruid
                        )
                    ) {
                        return true
                    }
                }
            }
            tmpScene.actions.forEach { action ->
                val rpcPayload = action.payload as? RpcPayload
                if (rpcPayload != null) {
                    if (findTip(
                            rpcPayload.deviceType,
                            rpcPayload.did,
                            rpcPayload.deviceName,
                            rpcPayload.ownerUid
                        )
                    ) {
                        return true
                    }
                }
            }
            return false
        }


        private fun findTip(
            deviceType: String?,
            did: String?,
            deviceName: String?,
            ownerUid: String?
        ): Boolean {
            if (deviceType == "car") {
                //如果车被删除了 不走浏览页
                //首页没有车房间 也没有room
                //判断是自己或者共驾人 创建的场景 如果room没了 就是车被删了
//
//                if (did != null) {
//                    val carRoom = CarManager.getCarRoomByCarDid(did)
//
//                }
                val carRooms = CarManager.getCarRooms().filter { carRoom -> carRoom.cardid == did }
                return carRooms.isEmpty()  || !carRooms[0].appear_home_list.contains(HomeManager.getInstance().currentHomeId)
            }
            return false
        }

        //地理围栏的判断
        fun locationError(tmpScene: SpecScene): Boolean {

            tmpScene.triggers.forEach { trigger ->
                val locationExtra = trigger.extra as? SpecScene.LocationExtra
                if (locationExtra != null) {
                    val curOfflinedevice = IdentifierManager.getOAID(
                        ServiceApplication.getAppContext()
                    )
                    val curUid = CoreApi.getInstance().miId
                    val device_uuid = locationExtra.device_uuid ?: ""
                    val locationUserId = locationExtra.uid ?:""


                    return if(device_uuid != ""){
                        if(curUid != locationUserId){//uid不一致
                            true
                        }
                        else{//uid一致看使用的手机
                            //当前手机
                            device_uuid != curOfflinedevice
                        }
                    } else{
                        //维持现状，如果只有老地理围栏，则管理员和创建者都能进入（仅副标题在有精确位置授权的时候，展示“该家庭内所有成员的任意手机”）
                        false
                    }
                }
            }

            return false
        }
        fun isXiaoaiEror(tmpScene: SpecScene):Boolean{
            return tmpScene.triggers.any { trigger ->
                trigger.extra is SpecScene.VoiceExtra
            }
        }
        fun isBrowseSpecScene(current:SpecScene):Boolean{
            val v1 = hasPerceptionOwnerScene(current)

            val v2 = v1 || isNecessaryError(current)

            //车车
            val v3 = v1 || v2 || carError(current)

            //地理围栏
            val v4 = v1 || v2 || v3 || locationError(current)
            return v1 || v2 || v3 || v4 || hasSoloDeviceNotIn(current, true)
        }

        /**
         * 个人设备的授权弹窗
         */
        fun showAuthDialogInMainThread(
            activity: BaseActivity,
            entity: HmAuthEntity,
            callback: ((Boolean, Boolean) -> Unit)?
        ) {
            if(!ServerCompact.isChinaMainLand(activity)) return
            CommonUtil.runMainThread {
                val api =
                    MiJiaRouter.getService(
                        IPeopleCarHomeDialog::class.java,
                        IPeopleCarHomeDialog.KEY
                    )
                api.showDialog(
                    entity,
                    activity,
                    true,
                    { wearSwitch, phoneSwitch, _, _, _ ->
                        CommonUtil.runMainThread {
                            SceneManagerV2.phoneAuth = if (phoneSwitch) 1 else 0
                            SceneManagerV2.watchAuth = if (wearSwitch) 1 else 0
                            callback?.invoke(wearSwitch, phoneSwitch)
                        }
                    },
                    {MiJiaLog.onlyLogcat("phone_scene","auth cancel")},
                    {CommonUtil.runMainThread { ToastUtil.show(R.string.smarthome_scene_change_switch_fail) } }
                )
            }
        }

        /**
         * 感知场景的授权弹窗
         */
        fun showPerceptionAuthDialog(
            activity: BaseActivity,
            entity: HmAuthEntity,
            forceNeedWearSwitch: Boolean,
            forceNeedPhoneSwitch: Boolean,
            callback: ((Boolean, Boolean) -> Unit)?
        ) {
            if (!ServerCompact.isChinaMainLand(activity)) return
            CommonUtil.runMainThread {
                val api = MiJiaRouter.getService(IPeopleCarHomeDialog::class.java, IPeopleCarHomeDialog.KEY)
                api.showDialog(entity, activity, true, forceNeedPhoneSwitch, forceNeedWearSwitch,
                    { wearSwitch, phoneSwitch, _, _, _ ->
                        CommonUtil.runMainThread {
                            SceneManagerV2.phoneAuth = if (phoneSwitch) 1 else 0
                            SceneManagerV2.watchAuth = if (wearSwitch) 1 else 0
                            callback?.invoke(wearSwitch, phoneSwitch)
                        }
                    },
                    { MiJiaLog.onlyLogcat("phone_scene", "auth cancel") },
                    { CommonUtil.runMainThread { ToastUtil.show(R.string.smarthome_scene_change_switch_fail) } }
                )
            }
        }
        /**
         * 人车家授权弹窗
         */
        fun checkRenCheJiaPermission(activity: BaseActivity,callBack: (isGranted: Boolean) -> Unit) {
            if (SceneManagerV2.isPhoneAuthGrant()) {
                checkMiAiPermission(callBack = callBack)
            } else {
                checkHmind2Api(activity) { wear_switch, phone_switch, entity ->
                    CommonUtil.runMainThread {
                        if (!phone_switch) {
                            showAuthDialogInMainThread(activity, entity) { wearSwitch, phoneSwitch ->
                                if (phoneSwitch) {
                                    checkMiAiPermission(callBack = callBack)
                                } else {
                                    callBack.invoke(false)
                                }
                            }
                        } else {
                            checkMiAiPermission(callBack = callBack)
                        }
                    }
                }
            }
        }
        fun checkHmind2Api(
            activity: Activity?,
            callback: ((Boolean, Boolean, HmAuthEntity) -> Unit)?
        ) {
            activity?:return
            if(!ServerCompact.isChinaMainLand(activity)) return
            val api =
                MiJiaRouter.getService(IPeopleCarHomeDialog::class.java, IPeopleCarHomeDialog.KEY)
            api.checkHmSwitch(activity,
                { wear_switch, phone_switch, hm_switch, if_data_auth_set, entity ->
                    CommonUtil.runMainThread {
//                        if (BuildConfig.DEBUG) {
//                            ToastUtil.show("check auth result: ${wear_switch}  ${phone_switch}")
//                        }
                        SceneManagerV2.phoneAuth = if (phone_switch) 1 else 0
                        SceneManagerV2.watchAuth = if (wear_switch) 1 else 0
                        callback?.invoke(wear_switch, phone_switch, entity)
                    }

                }) {
                CommonUtil.runMainThread { ToastUtil.show(R.string.smarthome_scene_change_switch_fail) }
            }
        }

        /**
         * 人车家授权弹窗感知
         */
        fun checkPerceptionRenCheJiaPermission(activity: BaseActivity,scId: Int, callBack: (isGranted: Boolean) -> Unit) {
            if (!ServerCompact.isChinaMainLand(CommonApplication.getAppContext())) return
            val privacyData = PersonalStatusManager.getPerceptionPrivacyData(scId)
            if (privacyData == null) {
                callBack.invoke(true)
                return
            }
            privacyData.apply {
                MiJiaLog.writeLogOnAll(LogType.GENERAL, "SpecSceneHelper", privacyData.toString())
                val needWearPermission = watch?.needPermission() ?: false
                val needPhonePermission = phone?.needPermission() ?: false
                val api = MiJiaRouter.getService(IPeopleCarHomeDialog::class.java, IPeopleCarHomeDialog.KEY)
                api.checkHmSwitch(activity,
                    { wearPermission, phonePermission, _, _, entity ->
                        CommonUtil.runMainThread {
                            SceneManagerV2.phoneAuth = if (phonePermission) 1 else 0
                            SceneManagerV2.watchAuth = if (wearPermission) 1 else 0
                            handlePermissionResult(
                                needWearPermission,
                                needPhonePermission,
                                wearPermission,
                                phonePermission,
                                callBack,
                                entity,
                                activity,
                            )
                        }
                    }) {
                    CommonUtil.runMainThread {
                        ToastUtil.show(R.string.smarthome_scene_change_switch_fail)
                    }
                }
            }
        }
        private fun ScenePrivacy.handlePermissionResult(
            needWearPermission: Boolean,
            needPhonePermission: Boolean,
            wearPermission: Boolean,
            phonePermission: Boolean,
            permissionCallback: (isGranted: Boolean) -> Unit,
            entity: HmAuthEntity,
            activity:BaseActivity
        ) {
            // 检查是否所有需要的权限都已经授予
            val allPermissionsGranted = wearPermission && phonePermission

            // 根据是否需要 wear 和 phone 权限进行处理
            when {
                // 需要 wear 和 phone 权限
                needWearPermission && needPhonePermission -> {
                    if (allPermissionsGranted) {
                        requestMiAiPermission(permissionCallback)
                    } else {
                        showPermissionDialog(activity,entity, needWearPermission, needPhonePermission) { wearSwitch, phoneSwitch ->
                            handlePermissionSwitch(
                                wearSwitch,
                                phoneSwitch,
                                needWearPermission,
                                needPhonePermission,
                                permissionCallback,
                            )
                        }
                    }
                }
                // 只需要 wear 权限
                needWearPermission -> {
                    if (wearPermission) {
                        permissionCallback.invoke(true)
                    } else {
                        showPermissionDialog(activity,entity, needWearPermission, needPhonePermission) { wearSwitch, phoneSwitch ->
                            handlePermissionSwitch(
                                wearSwitch,
                                phoneSwitch,
                                needWearPermission,
                                needPhonePermission,
                                permissionCallback
                            )
                        }
                    }
                }
                // 只需要 phone 权限
                needPhonePermission -> {
                    if (!phonePermission) {
                        showPermissionDialog(activity,entity, needWearPermission, needPhonePermission) { wearSwitch, phoneSwitch ->
                            handlePermissionSwitch(
                                wearSwitch,
                                phoneSwitch,
                                needWearPermission,
                                needPhonePermission,
                                permissionCallback
                            )
                        }
                    } else {
                        requestMiAiPermission(permissionCallback)
                    }
                }
                // 不需要任何权限
                else -> permissionCallback.invoke(true)
            }
        }

        // 显示权限对话框的函数
        private fun showPermissionDialog(
            activity:BaseActivity,
            entity: HmAuthEntity,
            needWearPermission: Boolean,
            needPhonePermission: Boolean,
            onSwitchChanged: (wearSwitch: Boolean, phoneSwitch: Boolean) -> Unit
        ) {
            showPerceptionAuthDialog(
                activity,
                entity,
                needWearPermission,
                needPhonePermission
            ) { wearSwitch, phoneSwitch ->
                onSwitchChanged(wearSwitch, phoneSwitch)
            }
        }

        // 处理权限开关的函数
        private fun ScenePrivacy.handlePermissionSwitch(
            wearSwitch: Boolean,
            phoneSwitch: Boolean,
            needWearPermission: Boolean,
            needPhonePermission: Boolean,
            permissionCallback: (isGranted: Boolean) -> Unit
        ) {
            when {
                needWearPermission && needPhonePermission -> {
                    if (wearSwitch && phoneSwitch) {
                        requestMiAiPermission(permissionCallback)
                    } else {
                        showPermissionDeniedToast(needWearPermission, needPhonePermission, wearSwitch, phoneSwitch)
                        permissionCallback.invoke(false)
                    }
                }

                // 只需要 wear 权限
                needWearPermission -> {
                    if (wearSwitch) {
                        permissionCallback.invoke(true)
                    } else {
                        showPermissionDeniedToast(needWearPermission, needPhonePermission, wearSwitch, phoneSwitch)
                        permissionCallback.invoke(false)
                    }
                }
                // 只需要 phone 权限
                needPhonePermission -> {
                    if (phoneSwitch) {
                        requestMiAiPermission(permissionCallback)
                    } else {
                        showPermissionDeniedToast(needWearPermission, needPhonePermission, wearSwitch, phoneSwitch)
                        permissionCallback.invoke(false)
                    }
                }
            }
        }

        private fun showPermissionDeniedToast(
            needWearPermission: Boolean,
            needPhonePermission: Boolean,
            wearPermission: Boolean,
            phonePermission: Boolean
        ) {
            // 根据权限需求和权限状态决定显示哪个toast
            val toastMessageResId = when {
                needPhonePermission && needWearPermission ->
                    if (!phonePermission && !wearPermission) R.string.Authorization_windows_for_phone_and_Miwear_verification_failed_toast
                    else if (!wearPermission) R.string.Authorization_windows_for_Miwear_verification_failed_toast
                    else if (!phonePermission) R.string.Authorization_windows_for_phone_verification_failed_toast
                    else null

                needWearPermission && !wearPermission -> R.string.Authorization_windows_for_Miwear_verification_failed_toast
                needPhonePermission && !phonePermission -> R.string.Authorization_windows_for_phone_verification_failed_toast
                else -> null
            }

            // 如果找到了toast消息，则显示它
            toastMessageResId?.let {
                ToastUtil.show(it)
            }
        }

        private fun ScenePrivacy.requestMiAiPermission(callBack: (isGranted: Boolean) -> Unit) {
            val iid = phone?.permission?.getOrNull(0)
            if (iid.isNullOrEmpty()) {
                callBack.invoke(true)
            } else {
                checkMiAiPermission(iid, callBack)
            }
        }

        private fun checkMiAiPermission(iid: String? = "", callBack: (isGranted: Boolean) -> Unit) {
            val key = if (iid.isNullOrEmpty()) {
                getIotSpecKey()
            } else {
                iid
            }
            val ps = PermissionCheck.checkPermission(CommonApplication.getAppContext(), key)
            if (ps != 0) {//弹权限
                val showState = PermissionCheck.requestPermission(CommonApplication.getAppContext(), key, ps)
                if (showState != 0) {
                    MiJiaLog.writeLogOnAll(LogType.SCENE, "phoneScene", "miai error $showState")
                }
                callBack.invoke(false)
            } else {
                callBack.invoke(true)
            }
        }

        private fun getIotSpecKey(): String {
            return "prop.2.13.2"
        }
    }


    interface ActivityResultCallback {
        fun onCall(
            activity: WeakReference<Activity>,
            resultCode: Int,
            requestCode: Int?,
            data: Intent?
        )
    }
}
=================================================================================
package com.xiaomi.smarthome.specscene.manager

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.text.TextUtils
import androidx.localbroadcastmanager.content.LocalBroadcastManager
import com.xiaomi.smarthome.application.CommonApplication
import com.xiaomi.smarthome.application.ServiceApplication
import com.xiaomi.smarthome.component.MiJiaRouter
import com.xiaomi.smarthome.constants.LoginConstants
import com.xiaomi.smarthome.device.SmartHomeDeviceHelper
import com.xiaomi.smarthome.device.SmartHomeDeviceManager
import com.xiaomi.smarthome.frame.core.CoreApi
import com.xiaomi.smarthome.globalsetting.GlobalSetting
import com.xiaomi.smarthome.homeroom.CarManager
import com.xiaomi.smarthome.homeroom.HomeManager
import com.xiaomi.smarthome.homeroom.device_order.mapToList
import com.xiaomi.smarthome.homeroom.model.Home
import com.xiaomi.smarthome.homeroom.model.Home.PERMIT_HOME_MEMBER
import com.xiaomi.smarthome.homeroom.model.Home.PERMIT_HOME_OWNER
import com.xiaomi.smarthome.homeroom.model.IHomeManagerApi
import com.xiaomi.smarthome.library.log.LogType
import com.xiaomi.smarthome.library.log.MiJiaLog
import com.xiaomi.smarthome.scene.ConstantType
import com.xiaomi.smarthome.scene.activity.CommonSceneOnline
import com.xiaomi.smarthome.scene.api.RemoteSpecSceneApi.SCENE_TPLV2_QUERY_DID
import com.xiaomi.smarthome.scene.api.RemoteSpecSceneApi.SCENE_TPLV2_QUERY_HOME
import com.xiaomi.smarthome.scene.api.RemoteSpecSceneApi.SCENE_TPLV2_QUERY_USER
import com.xiaomi.smarthome.scene.data.SpecSceneDataSource
import com.xiaomi.smarthome.scene.data.SpecSceneDataSource.writeTCAConfigCachePerson
import com.xiaomi.smarthome.specscene.bean.ATplItem
import com.xiaomi.smarthome.specscene.bean.TCTplItem
import com.xiaomi.smarthome.specscene.manager.Tplv2Provider.TPL_EFFECTIVE_TIME
import com.xiaomi.smarthome.specscene.manager.Tplv2Provider.TPL_REPEAT_EFFECTIVE_TIME
import com.xiaomi.smarthome.specscene.personal.viewmodel.PersonalStatusManager
import com.xiaomi.smarthome.specscene.personal.viewmodel.QuickActionManager
import com.xiaomi.smarthome.specscene.personal.viewmodel.XiaoaiDeviceManager
import com.xiaomi.smarthome.specscene.util.SpecSceneHelper
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.GlobalScope
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import org.json.JSONArray
import org.json.JSONObject
import java.util.concurrent.ConcurrentHashMap
import java.util.concurrent.CopyOnWriteArrayList
import java.util.concurrent.atomic.AtomicBoolean

object SpecSceneConfigRepository {
    private const val TAG_CALLBACK = "bussiness-lifesycle"
    const val TAG_REQUEST = "request-tag"
    private var curPhoneId: String? = null

    init {
        curPhoneId = SmartHomeDeviceHelper.getCurPhoneDid()
    }

    /**
     * 存储信息
     */
    //家庭-房间
    //房间-设备
    //设备-tpl列表映射
    val actionsDid: ConcurrentHashMap<String, CopyOnWriteArrayList<Int>> = ConcurrentHashMap()//设备id, 配置id
    val triggersDid: ConcurrentHashMap<String, CopyOnWriteArrayList<Int>> = ConcurrentHashMap()//设备id, 配置id
    val conditionsDid: ConcurrentHashMap<String, CopyOnWriteArrayList<Int>> = ConcurrentHashMap()//设备id, 配置id
    //设备-std-tpl列表映射
    val actionStdDid: ConcurrentHashMap<String, CopyOnWriteArrayList<String>> = ConcurrentHashMap()//设备id, 配置id
    val triggerStdDid: ConcurrentHashMap<String, CopyOnWriteArrayList<String>> = ConcurrentHashMap()//设备id, 配置id
    val conditionStdDid: ConcurrentHashMap<String, CopyOnWriteArrayList<String>> = ConcurrentHashMap()//设备id, 配置id
    //tpl_id-tpl条目映射
    val actionsOnline: ConcurrentHashMap<Int, ATplItem> = ConcurrentHashMap()//配置id, 配置详情
    val triggersOnline: ConcurrentHashMap<Int, TCTplItem> = ConcurrentHashMap()//配置id, 配置详情
    val conditionsOnline: ConcurrentHashMap<Int, TCTplItem> = ConcurrentHashMap()//配置id, 配置详情
    //标准库id-tpl条目映射
    val actionsStd: ConcurrentHashMap<String, ATplItem> = ConcurrentHashMap()//配置id, 标准化表，厂商表中有的，这里可能没有
    val triggersStd: ConcurrentHashMap<String, TCTplItem> = ConcurrentHashMap()//配置id, 标准化表，厂商表中有的，这里可能没有
    val conditionsStd: ConcurrentHashMap<String, TCTplItem> = ConcurrentHashMap()//配置id, 标准化表，厂商表中有的，这里可能没有

    val custom2Std:ConcurrentHashMap<Int, String> = ConcurrentHashMap()
    private val receiver = object:BroadcastReceiver() {
        override fun onReceive(context: Context?, intent: Intent?) {
            when (intent?.action) {
                LoginConstants.ACTION_ON_LOGOUT -> clear(intent.action ?: "log out")
                "home_room_updated" -> {
                    //切换家庭，有时发这个广播，有时不发。为什么？
                    if (TextUtils.equals(
                            "home_room_sync",
                            intent.getStringExtra("operation")
                        )
                    ) {
                        refresh(intent.action ?: "")
                    }
                }
                "home_room_home_changed" -> {
                    //切换家庭，发这个广播。但是不一定home ready了
                    SceneManagerV2.getInstance().initCommonOnlineScenes(arrayListOf())
                    isRefreshing.set(false)
                    refresh(intent.action ?: "")
                }
            }
        }
    }
    init {
        MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_CALLBACK, "specscene config repository load called")
        val intentFilter = IntentFilter().apply {
            addAction(LoginConstants.ACTION_ON_LOGOUT)
            addAction(LoginConstants.ACTION_ON_LOGIN_SUCCESS)
            addAction("home_room_updated")
            addAction("home_room_home_changed")
        }
        LocalBroadcastManager.getInstance(CommonApplication.getAppContext())
            .registerReceiver(receiver, intentFilter)
    }
    private val homeApi by lazy {
        MiJiaRouter.getService(
            IHomeManagerApi::class.java,
            IHomeManagerApi.KEY_HOME
        )
    }
    private var lastRequestTime: MutableMap<String, Long> = mutableMapOf()
    private var isRefreshing = AtomicBoolean(false)
    private val configProfiles: MutableMap<String, Long> = mutableMapOf()
    private suspend fun getDateFromCache(homeId:String?=null) {
        val tplCache = SpecSceneDataSource.readTCAConfigCache(homeId?:"")
        MiJiaLog.writeLogOnGrey(LogType.SCENE,
            TAG_REQUEST,
            "read cache sucess ${tplCache.first}    ${tplCache.second.length()}"
        )
        synchronized(this) {
            val jsonList = tplCache.second.mapToList(
                arrayListOf()
            ) {
                it as JSONObject
            }
            SceneManagerV2.getInstance().initCommonOnlineScenes(jsonList)
            if (tplCache.first > getCurrentHomeLastRequestTime(homeId)) {
                parseData(tplCache.second, tplCache.first)
            }
        }
    }
    suspend fun refreshSoloDeviceConfig() {
        MiJiaLog.writeLogOnGrey(
            LogType.SCENE, TAG_REQUEST,
            "---start refresh solo config---"
        )
        val currentTs = System.currentTimeMillis()
        val personTCA = SpecSceneDataSource.getHMConfigFlow(curPhoneId)
        MiJiaLog.writeLogOnGrey(
            LogType.SCENE, TAG_REQUEST,
            "---end getHMConfigFlow---${personTCA.second.length()}---"
        )
        personTCA.second.also { ja ->
            if (ja.length() > 0) {
                for (i in 0 until ja.length()) {
                    ja.optJSONObject(i)?.optString("did")?.also { did ->
                        configProfiles[did] = currentTs
                    }
                }
                parseData(ja, currentTs)
            }
        }
        personTCA.third.also { ja ->
            if (ja.length() > 0) {
                for (i in 0 until ja.length()) {
                    ja.optJSONObject(i)?.optString("did")?.also { did ->
                        configProfiles[did] = currentTs
                    }
                }
                parseData(ja, currentTs)
            }
        }

    }
    fun refreshOtherSceneConfig(){
        fun realRefresh(requestShowHome:Home){

            val currentTs = System.currentTimeMillis()
            GlobalScope.launch(Dispatchers.IO)  {
                if(!CoreApi.getInstance().isInternationalServer && !GlobalSetting.IS_GooglePlay_CHANNEL){
                    PersonalStatusManager.getAllData(requestShowHome.id.toLong(), requestShowHome.ownerUid)
                }

                QuickActionManager.getAllData(requestShowHome.id.toLong(), requestShowHome.ownerUid)
                XiaoaiDeviceManager.getAllXiaoaiData(requestShowHome.id.toLong(), "")
                try {
                    val personTCA = SpecSceneDataSource.getHMConfigFlow(curPhoneId)
                    writeTCAConfigCachePerson(
                        CoreApi.getInstance().miId,
                        System.currentTimeMillis(),
                        arrayListOf<JSONObject>().apply {
                            for (i in 0 until personTCA.first.length()) {
                                add(personTCA.first.optJSONObject(i))
                            }
                        })
                } catch (e: Exception) {
                    MiJiaLog.writeLogOnAll(LogType.SCENE, TAG, "personTCA: e" + e.message)
                }
                try {
                    getPersonDisplayList(requestShowHome.ownerUid.toString())?.let {
                        SpecSceneDataSource.writeCachePersonMainItem(requestShowHome.id, currentTs,
                            it
                        )
                    }

                } catch (e: Exception) {
                    MiJiaLog.writeLogOnAll(LogType.SCENE,TAG, "totalJaPersonMainItem: e"+e.message)
                }
                try {
                    SpecSceneDataSource.writeCacheLightItem(requestShowHome.id, currentTs, getModuleDevices(requestShowHome.id,requestShowHome.ownerUid,"light").also {
                        MiJiaLog.writeLogOnAll(LogType.SCENE,TAG,"updateData: totalLightItem $it")

                    })
                } catch (e: Exception) {
                    MiJiaLog.writeLogOnAll(LogType.SCENE,TAG, "totalLightItem: ${e.message}")
                }
                try {
                    SpecSceneDataSource.writeCacheCurtainItem(requestShowHome.id, currentTs, getModuleDevices(requestShowHome.id,requestShowHome.ownerUid,"curtain").also {
                        MiJiaLog.writeLogOnAll(LogType.SCENE,TAG, "updateData: totalCurtainItem $it")
                    })

                } catch (e: Exception) {
                    MiJiaLog.writeLogOnAll(LogType.SCENE,TAG, "totalCurtainItem: e"+e.message)
                }
                QuickActionManager.getAllData(requestShowHome.id.toLong(), requestShowHome.ownerUid)
            }
        }
        val curHome = homeApi.currentHome
        if(curHome == null){
            MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST, "refreshOtherSceneConfig home is not ready.waiting for home ready... ")
            LocalBroadcastManager.getInstance(CommonApplication.getAppContext())
                .registerReceiver(object : BroadcastReceiver() {
                    override fun onReceive(context: Context?, intent: Intent?) {
                        if (HomeManager.KEY_OPERATION_SYNC == intent?.getStringExtra(HomeManager.KEY_OPERATION)) {
                            MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST, "refreshOtherSceneConfig home is ready")
                            val tmpHome = homeApi.currentHome ?: return
                            LocalBroadcastManager.getInstance(CommonApplication.getAppContext())
                                .unregisterReceiver(this)
                            realRefresh(tmpHome)

                        }
                    }

                }, IntentFilter(HomeManager.ACTION_HOME_ROOM_UPDATED))
        }else{
            realRefresh(curHome)
        }

    }
    fun refresh(cause:String="",deltaDids:List<String>?=null){
        if (isRefreshing.get()) return
        MiJiaLog.writeLogOnAll(LogType.SCENE,TAG_CALLBACK, "refresh: cause by = $cause =")
        val curHome = homeApi.currentHome
        if (curHome == null) {
            MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST, "home is not ready.waiting for home ready... ")
            LocalBroadcastManager.getInstance(CommonApplication.getAppContext())
                .registerReceiver(object : BroadcastReceiver() {
                    override fun onReceive(context: Context?, intent: Intent?) {
                        if (HomeManager.KEY_OPERATION_SYNC == intent?.getStringExtra(HomeManager.KEY_OPERATION)) {
                            MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST, "home is ready")
                            val tmpHome = homeApi.currentHome ?: return
                            LocalBroadcastManager.getInstance(CommonApplication.getAppContext())
                                .unregisterReceiver(this)
                            GlobalScope.launch(Dispatchers.IO)  {
                                getDateFromCache(tmpHome.id)
                                withContext(Dispatchers.Main) {
                                    Tplv2Provider.updateData(JSONArray(), true)
                                }
                            }

                            updateData(tmpHome, cause, deltaDids)
                        }
                    }

                }, IntentFilter(HomeManager.ACTION_HOME_ROOM_UPDATED))
        } else {
            val hid = curHome.id
            GlobalScope.launch(Dispatchers.IO) {
                getDateFromCache(hid)
                //这里是去请求tca的接口
                withContext(Dispatchers.Main) {
                    Tplv2Provider.updateData(JSONArray(), true)
                }
            }
            updateData(curHome, cause, deltaDids)
        }
    }
    /**
     * 加载硬盘缓存到内存-通知UI
     * 计算差异-更新内存-通知UI
     */
    private fun updateData(requestShowHome:Home,cause: String,dids:List<String>?=null) {
        MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST, "$cause request update ${System.currentTimeMillis()}")
        if (requestShowHome.permitLevel <= PERMIT_HOME_MEMBER) {//2
            return
        }
        if (isRefreshing.get()) return
        isRefreshing.set(true)
        SpecSceneHelper.sceneCheckUserGray(ServiceApplication.getAppContext(),null)
        val currentTs = System.currentTimeMillis()
        if (dids?.isEmpty() == true && currentTs - getCurrentHomeLastRequestTime() < TPL_REPEAT_EFFECTIVE_TIME) {//防止连续重复访问
            MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST, "in legal ts_window")
            sendRefreshOver(requestShowHome,"cache")
            isRefreshing.set(false)
            return
        }
        curPhoneId = SmartHomeDeviceHelper.getCurPhoneDid()
        val soloDevices = SmartHomeDeviceManager.getInstance().findSoloDevices()
        val deltaDids = mutableSetOf<String>()//计算当前家庭+车家庭
        if(dids?.isNotEmpty() == true){
            deltaDids.addAll(dids)
        }
        val soloDids = mutableSetOf<String>()//计算无主设备
//        val deltaParams = mutableListOf<Triple<String, Long, List<String>?>>()
        var queryType = SCENE_TPLV2_QUERY_USER//默认请求账号下全部
        val queryParam = mutableListOf<Triple<String, Long, List<String>?>>()
        soloDevices.forEach {
            if (currentTs - (configProfiles[it.did] ?: 0) > TPL_EFFECTIVE_TIME) {
                it.did?.also { soloDid -> soloDids.add(soloDid) }
            }
        }
        if (getCurrentHomeLastRequestTime() <= 0L) {
            MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST, "current home ${HomeManager.getInstance().currentHome.rawName} first request tpl")
            queryType = SCENE_TPLV2_QUERY_USER
            queryParam.apply {
                add(Triple(requestShowHome.id, requestShowHome.ownerUid, null))
                if (homeApi.currentHome.permitLevel >= PERMIT_HOME_OWNER) {
                    addAll(
                        CarManager.getCarRooms()
                            .filter { it.homeID != null && it.uid != null && it.dids?.isNotEmpty() == true }
                            .map { Triple(it.homeID!!, it.uid!!.toLong(), null) })
                    CarManager.getCarRooms()
                        .filter { carRoomData -> carRoomData.temporaryDids?.isNotEmpty() == true }
                        .map {
                            it.temporaryDids?.forEach { tdid ->
                                homeApi.getHomeByDid(tdid)?.also { th ->
                                    add(Triple(th.id, th.ownerUid, null))
                                }
                            }
                        }
                }
            }

            MiJiaLog.writeLogOnAll(LogType.SCENE, TAG_REQUEST,"solodevices: ${soloDevices.size}   soloDids: ${soloDids.size}")
        } else {
            queryType = SCENE_TPLV2_QUERY_DID
            CarManager.getCarRooms().forEach {
                if (currentTs - (configProfiles[it.cardid] ?: 0) > TPL_EFFECTIVE_TIME) {
                    it.cardid?.also { carDid -> deltaDids.add(carDid) }
                }
                it.temporaryDids?.forEach { tDid ->
                    if (currentTs - (configProfiles[tDid] ?: 0) > TPL_EFFECTIVE_TIME) {
                        deltaDids.add(tDid)
                    }
                }
                it.dids?.forEach { carIotDid ->
                    if (currentTs - (configProfiles[carIotDid]
                            ?: 0) > TPL_EFFECTIVE_TIME
                    ) {
                        deltaDids.add(carIotDid)
                    }
                }
            }

            MiJiaLog.writeLogOnAll(LogType.SCENE, TAG_REQUEST,"SCENE_TPLV2_QUERY_DID solodevices: ${soloDevices.size}   soloDids: ${soloDids.size}")
            deltaDids.addAll(requestShowHome.allDids.filter {
                currentTs - (configProfiles[it] ?: 0) > TPL_EFFECTIVE_TIME
            }.map { it })
            if (deltaDids.isEmpty() && soloDids.isEmpty()) {
                sendRefreshOver(requestShowHome,cause)
                MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST, "all did is new config")
                isRefreshing.set(false)
                return
            }

            MiJiaLog.writeLogOnAll(LogType.SCENE,TAG_REQUEST,  "deltaDid size: ${deltaDids.size}")
            val didHomeMap = HomeManager.getInstance().didHomeMap
            deltaDids.groupBy {
                didHomeMap[it]?.id
            }.filter {
                it.key != null
            }.forEach {
                homeApi.getHomeById(it.key)
                    ?.also { home ->
                        queryParam.add(Triple(home.id, home.ownerUid, it.value))
                    }
            }
            deltaDids.groupBy {
                CarManager.getCarRoomByDid(
                    it,
                    inDids = true,
                    inCars = true,
                    inTemporaryDevice = false
                )
            }.filter { it.key != null }.forEach { homeGroup ->
                if (homeGroup.key != null && homeGroup.value.isNotEmpty()) {
                    val chunkedThird = if (homeGroup.value.size > 300) {
                        homeGroup.value.chunked(300)
                    } else {
                        arrayListOf<List<String>>().apply { add(homeGroup.value) }
                    }
                    chunkedThird.forEach { page ->
                        queryParam.add(
                            Triple(
                                homeGroup.key?.homeID ?: "",
                                homeGroup.key?.uid?.toLong() ?: 0L,
                                page
                            )
                        )
                    }

                }
            }

        }


        if (queryParam.isEmpty() && soloDids.isEmpty()) {
            sendRefreshOver(requestShowHome,"cache")
            MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST,"did query param is null")
            isRefreshing.set(false)
            return
        }
        MiJiaLog.writeLogOnAll(LogType.SCENE,TAG_REQUEST, "did query param is not null ${queryParam.size} ${soloDids.size}")
        queryParam.forEach {
            MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST, "did query param:${it.first}  ${it.second}  ${it.third?.size}  ")
        }
        val totalJa = arrayListOf<JSONObject>()

        GlobalScope.launch(Dispatchers.IO)  {
            var page = 0
            val requestSuccess = AtomicBoolean(true)
            val subList = if (queryParam.size > 10) {
                MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST, "need ${queryParam.size} > 10 need chunck")
                queryParam.chunked(10)
            } else {
                arrayListOf<List<Triple<String, Long, List<String>?>>>().apply { add(queryParam) }
            }
            MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST, "need ${subList.size} page(s)")
            if (soloDids.isNotEmpty()) {
                MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST,
                    "---start getHMConfigFlow---"
                )
                val personTCA = SpecSceneDataSource.getHMConfigFlow(curPhoneId)
                MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST,
                    "---end getHMConfigFlow---${personTCA.second.length()}---"
                )
                personTCA.second.also { ja ->
                    if (ja.length() > 0) {
                        for(i in 0 until ja.length()){
                            ja.optJSONObject(i)?.optString("did")?.also{did->
                                configProfiles[did] = currentTs
                            }
                        }
                        parseData(ja, currentTs)
                        ja.mapToList(totalJa) { it as JSONObject }
                    }
                }
                personTCA.third.also { ja ->
                    if (ja.length() > 0) {
                        for(i in 0 until ja.length()){
                            ja.optJSONObject(i)?.optString("did")?.also{did->
                                configProfiles[did] = currentTs
                            }
                        }
                        parseData(ja, currentTs)
                        ja.mapToList(totalJa) { it as JSONObject }
                    }
                }

            }
            for (sunHomeParam in subList) {
                if (sunHomeParam.isEmpty()) {
                    MiJiaLog.onlyLogcat(TAG_REQUEST, "sunHomeParam is empty")
                    continue
                }
                MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_REQUEST, "--${sunHomeParam.size}")
                sunHomeParam.forEach { oneHomeParam ->

                    MiJiaLog.onlyLogcat(
                        TAG_REQUEST,
                        "start query ${if (queryType == SCENE_TPLV2_QUERY_USER) "SCENE_TPLV2_QUERY_USER" else if (queryType == SCENE_TPLV2_QUERY_DID) "SCENE_TPLV2_QUERY_DID" else "UNKNWON"}  ${oneHomeParam.first}  ${
                            if (SpecSceneHelper.isCarHome(oneHomeParam.first)) CarManager.getCarRoomByHomeId(
                                oneHomeParam.first
                            )?.name ?: "car home" else HomeManager.getInstance()
                                .getHomeById(oneHomeParam.first)?.rawName
                        } ${oneHomeParam.third?.size ?: 0}"
                    )
                    oneHomeParam.third?.forEach {
                        MiJiaLog.onlyLogcat(
                            TAG_REQUEST,
                            "---$oneHomeParam---${SpecSceneHelper.findDeviceById(it)?.name ?: ""}---"
                        )
                    }
                }

                getTplV2Flow(
                    queryType = queryType,
                    homeList = sunHomeParam
                ).collect { tplAndFilter ->
                    if (tplAndFilter.first == null && tplAndFilter.second == null) {
                        requestSuccess.set(false)
                        return@collect
                    }
                    tplAndFilter.first?.also { ja ->
                        if (ja.length() > 0) {
                            parseData(ja, currentTs)
                        }
                        ja.mapToList(totalJa) { it as JSONObject }
//                        if (page == 0) SceneManagerV2.getInstance()
//                            .initSceneCompatibleList(tplAndFilter.second)
                        page++
                    }

                }

            }
            if (requestSuccess.get()) {
                SpecSceneDataSource.writeTCAConfigCache(requestShowHome.id, currentTs, totalJa)
                SceneManagerV2.getInstance().initCommonOnlineScenes(totalJa)
            }
            withContext(Dispatchers.Main) {
                if (requestSuccess.get()) {
                    when (queryType) {
                        SCENE_TPLV2_QUERY_USER -> {
                            HomeManager.getInstance().allHome.forEach {
                                if(it.isOwner || it.isManager){
                                    lastRequestTime[it.id] = currentTs
                                    it.allDids.forEach {tdid->configProfiles[tdid] = currentTs  }
                                }
                            }
                        }
                        SCENE_TPLV2_QUERY_HOME -> {
                            lastRequestTime[requestShowHome.id] = currentTs
                            requestShowHome.allDids.forEach {
                                MiJiaLog.writeLogOnGrey(
                                    LogType.SCENE,
                                    TAG_REQUEST,"all did: $it  ${SmartHomeDeviceManager.getInstance().getDeviceByDid(it).name}")
                                configProfiles[it] = currentTs
                            }
                        }
                        SCENE_TPLV2_QUERY_DID -> {
                            lastRequestTime[requestShowHome.id] = currentTs
                            deltaDids.forEach {
                                configProfiles[it] = currentTs

                            }
                        }
                    }
                    MiJiaLog.writeLogOnGrey(
                        LogType.SCENE,
                        TAG_REQUEST,
                        "from server success $queryType .size is ${totalJa.size}  ${ requestShowHome.allDids.size}  ${deltaDids.size}"
                    )
                }
                sendRefreshOver(requestShowHome, if (requestSuccess.get()) "server" else "cache")
                Tplv2Provider.updateData(JSONArray(), true)
                isRefreshing.set(false)
            }
        }
    }
    private fun sendRefreshOver(h: Home?,from:String) {
        GlobalScope.launch(Dispatchers.Main) {
            Tplv2Provider.sendBroadCast(
                Tplv2Provider.ACTION_TPLV2_FROM_SERVER,
                hashMapOf<String, String>().apply {
                    put(Tplv2Provider.KEY_TPLV2_HOME_ID, h?.id ?: "")
                })
        }
    }
    private fun getCurrentHomeLastRequestTime(homeId:String? = HomeManager.getInstance().currentHomeId):Long{
        homeId?:return 0L
        return lastRequestTime[homeId] ?: 0L
    }

    private fun parseData(configs: JSONArray, ts: Long) {
        val canNotBeCondition: Set<Int> = setOf(101,103, 104, 108)
//        val specialTypeCTIDs: Set<Int> = setOf(103, 302)
        for (i in 0 until configs.length()) {
            val didConfigs = configs.optJSONObject(i)
            val did = didConfigs.optString("did")
            val model = didConfigs.optString("model")
            val launchConfig = didConfigs.optJSONObject("value")?.optJSONArray( "launch")
            val actionConfig = didConfigs.optJSONObject("value")?.optJSONArray("action_list")
            if (!triggersDid.contains(did)) {
                triggersDid[did] = CopyOnWriteArrayList()
            } else {
                triggersDid[did]?.clear()
            }
            if (!triggerStdDid.contains(did)) {
                triggerStdDid[did] = CopyOnWriteArrayList()
            } else {
                triggerStdDid[did]?.clear()
            }
            if (!conditionsDid.contains(did)) {
                conditionsDid[did] = CopyOnWriteArrayList()
            } else {
                conditionsDid[did]?.clear()
            }
            val dTCTplItemsBefore = mutableListOf<Int>()
            val dTCTplItemsAfter = mutableListOf<Int>()
            for (j in 0 until (launchConfig?.length() ?: 0)) {
                val launch = TCTplItem.optItem(launchConfig?.optJSONObject(j),model, did, null, null, 1)
                if ((launch.mAttr is CommonSceneOnline.SceneAttrFencing) && TextUtils.equals(
                        did,
                        curPhoneId
                    )
                ) {
                    if (!SpecSceneHelper.isSupportGeofence) {//这样判断不严谨：米家判断是否支持，看米家的地理位置权限以及米家中地理围栏服务SDK是否初始化失，这里仅弥补miai没有上报动态能力的case
                        continue
                    }
                }
                if(launch.id == 0){
                    MiJiaLog.onlyLogcat("jmd236","===trigger id empty $did ${launch.stdId}")
                    if(launch.stdId.isNullOrEmpty()){
                        continue
                    }
                    triggersStd[launch.stdId!!] = launch
                    triggerStdDid[did]?.add(launch.stdId)
                    if(launch.relatedId>0)
                        custom2Std[launch.relatedId] = launch.stdId!!
                }else{
                    triggersOnline[launch.id] = launch
                }

                if (!canNotBeCondition.contains(launch.mCompatibleId)) {
                    val condition = TCTplItem.optItem(launchConfig?.optJSONObject(j),model, did, null, null, 1).apply {
                        mName = mConditionName
                    }
                    if(condition.id == 0){
                        if(condition.stdId.isNullOrEmpty()){
                            continue
                        }
                        conditionsStd[condition.stdId!!] = launch
                        conditionStdDid[did]?.add(condition.stdId)
                        if(launch.relatedId>0)
                            custom2Std[launch.relatedId] = launch.stdId!!
                    }else{
                        conditionsOnline[condition.id] = condition
                    }
                }
                //TODO:标准化，标准化的时候，这里要去掉，排序在展示层做
                if (launch.rank < ConstantType.DEFAULT_TPL_RANK) {
                    dTCTplItemsBefore.add(launch.id)
                } else if (launch.rank > ConstantType.DEFAULT_TPL_RANK) {
                    dTCTplItemsAfter.add(launch.id)
                } else {
                    triggersDid[did]?.add(launch.id)
                }
            }
            MiJiaLog.onlyLogcat("jmd236","$did  ${dTCTplItemsBefore.size}   ${dTCTplItemsAfter.size}  ${triggersDid[did]?.size}")
            if (dTCTplItemsBefore.isNotEmpty()) triggersDid[did]?.addAll(
                0,
                dTCTplItemsBefore
            )
            if (dTCTplItemsAfter.isNotEmpty()) triggersDid[did]?.addAll(
                dTCTplItemsAfter
            )
            MiJiaLog.onlyLogcat("jmd236","$did  ${triggersDid[did]?.size}")
            triggersDid[did]?.filter {
                !canNotBeCondition.contains(conditionsOnline[it]?.mCompatibleId ?: -1)
            }?.also {
                conditionsDid[did]?.addAll(it)
            }

            if (!actionsDid.contains(did)) {
                actionsDid[did] = CopyOnWriteArrayList()
            } else {
                actionsDid[did]?.clear()
            }
            if (!actionStdDid.contains(did)) {
                actionStdDid[did] = CopyOnWriteArrayList()
            } else {
                actionStdDid[did]?.clear()
            }
            val dATplItemsBefore = mutableListOf<Int>()
            val dATplItemsAfter = mutableListOf<Int>()
            for (j in 0 until (actionConfig?.length() ?: 0)) {
                val action = ATplItem.optItem(actionConfig?.optJSONObject(j), did, null, null, 1)
                if(action.id == 0){
                    MiJiaLog.onlyLogcat("jmd236","===action id empty $did ${action.stdId}")
                    if(action.stdId.isNullOrEmpty()){
                        continue
                    }
                    actionsStd[action.stdId!!] = action
                    actionStdDid[did]?.add(action.stdId)
                    if (action.relatedId > 0)
                        custom2Std[action.relatedId] = action.stdId!!
                }else{
                    actionsOnline[action.id] = action
                }

                if (action.rank < ConstantType.DEFAULT_TPL_RANK) {
                    dATplItemsBefore.add(action.id)
                } else if (action.rank > ConstantType.DEFAULT_TPL_RANK) {
                    dATplItemsAfter.add(action.id)
                } else {
                    actionsDid[did]?.add(action.id)
                }
            }
            if (dATplItemsBefore.isNotEmpty()) actionsDid[did]?.addAll(
                0,
                dATplItemsBefore
            )
            if (dATplItemsAfter.isNotEmpty()) actionsDid[did]?.addAll(
                dATplItemsAfter
            )
        }
    }
    fun clear(cause: String = "") {
        MiJiaLog.writeLogOnGrey(LogType.SCENE,TAG_CALLBACK, "clear $cause")
        //数据清空
        isRefreshing.set(false)
        actionsDid.clear()
        triggersDid.clear()
        conditionsDid.clear()
        //tpl_id-tpl条目映射
        actionsOnline.clear()
        triggersOnline.clear()
        conditionsOnline.clear()
        configProfiles.clear()
        lastRequestTime.clear()
        SceneManagerV2.getInstance().initCommonOnlineScenes(arrayListOf())
    }
    fun unregisterReceiver() {
        LocalBroadcastManager.getInstance(CommonApplication.getAppContext())
            .unregisterReceiver(receiver)
        clear("destroySceneManager")
    }

    fun load() {
        MiJiaLog.onlyLogcat(TAG_CALLBACK, "specscene config repository load called")
        val intentFilter = IntentFilter().apply {
            addAction(LoginConstants.ACTION_ON_LOGOUT)
            addAction(LoginConstants.ACTION_ON_LOGIN_SUCCESS)
            addAction("home_room_updated")
            addAction("home_room_home_changed")
        }
        LocalBroadcastManager.getInstance(CommonApplication.getAppContext())
            .registerReceiver(receiver, intentFilter)
    }

    /**
     * 分页获取tplv2模板（厂商配置）
     */
    private suspend fun getTplV2Flow(
        queryType: Int = SCENE_TPLV2_QUERY_USER,
        homeList: List<Triple<String, Long, List<String>?>>? = null
    ): Flow<Pair<JSONArray?, JSONArray?>> {
        return SpecSceneDataSource.getTCAConfigFlow(
            queryType = queryType,
            homeList = homeList,
            limit = 300
        )
    }
    private suspend fun getPersonDisplayList(uid:String): List<JSONObject>? {
        return SpecSceneDataSource.getPersonDisplayList(uid)
    }
    private suspend fun getModuleDevices(hid: String, ownerUid: Long, module: String): List<JSONObject>{
        return SpecSceneDataSource.getModuleDevices(hid ,ownerUid, module)
    }


    suspend fun requestTplByDid(did: String, homeId: String, homeOwner: Long): List<Int>? {
        val currentTs = System.currentTimeMillis()
        val lastTs = configProfiles[did]
        if (currentTs - (lastTs ?: 0) < 60 * 1000L && actionsDid[did] != null) {
            return actionsDid[did]
        }
        val result = mutableListOf<Int>()
        getTplV2Flow(queryType = SCENE_TPLV2_QUERY_DID,
            homeList = arrayListOf<Triple<String, Long, List<String>?>>().apply {
                Triple(homeId, homeOwner, arrayListOf<String>().apply { add(did) })
            }).collect { tplAndFilter ->
            tplAndFilter.first?.also { ja ->
                parseData(ja, currentTs)
            }
        }
        return result
    }
}

========================================================================================
package com.xiaomi.smarthome.specscene.bean

import android.text.TextUtils
import com.xiaomi.smarthome.device.api.spec.instance.Spec
import com.xiaomi.smarthome.device.api.spec.instance.SpecDevice
import com.xiaomi.smarthome.library.log.MiJiaLog
import com.xiaomi.smarthome.scene.ConstantType
import com.xiaomi.smarthome.scene.activity.CommonSceneOnline
import com.xiaomi.smarthome.scene.rec.SpecActionParam
import com.xiaomi.smarthome.scene.rec.SpecPropertyParam
import com.xiaomi.smarthome.specscene.manager.SceneManagerV2
import com.xiaomi.smarthome.specscene.util.MultiSwitchTitleHelper
import org.json.JSONArray
import org.json.JSONObject

class ATplItem(
    did: String?,//设备did
    homeId: String?,//所属家庭
    roomId: String?,//所属房间
    mName: String?,//名称
    id: Int,//唯一标识
    stdSaId:String?,
    stdRuleId:Int,
    relatedScId:Int,
    mGroupId: Int,//属于某个聚合结构
    mGroupName: String?,//聚合结构的名称
    var mCommand: String? = null,//自动化指令
    var mValue: Any? = null,//自动化参数
    var mParamAction: String? = null,//跳转参数
    mCompatibleId: Int? = 0,//互斥码
    mAttr: CommonSceneOnline.SceneAttr? = null,//选项描述
    mShadow:ShadowTCAItem?,
    isSpecPicker: Boolean = false,//标识spec滚轴，准备回填参数
    protocolType: Int? = 2,//协议类型
    from: Int,//配置来源
    rank: Int = ConstantType.DEFAULT_TPL_RANK//排序相关字段
) : TCABaseItem(did,homeId,roomId,mName,id,stdSaId,stdRuleId,relatedScId,mCompatibleId,mGroupId, mGroupName,mAttr,mShadow,isSpecPicker,from,rank,protocolType) {
    companion object {

        fun optItem(json: JSONObject?, did: String?, hid: String?, rid: String?,from:Int): ATplItem {
            val payloadJsonObj = json?.optJSONObject("payload")
            val isNumberPicker: Boolean =
                    payloadJsonObj?.optJSONObject("attr")?.optInt("attr_id") ?: payloadJsonObj?.optJSONObject("attr_new")
                    ?.optInt("attr_id") ?: 0 == 2001
            val isSpec = isSpec(payloadJsonObj?.opt("value"))
            val value = if (isNumberPicker && isSpec) {
                when {
                    payloadJsonObj?.optJSONObject("value")?.has("piid") == true -> {
                        SpecPropertyParam.parseFromSmarthomeScene(
                            json.optJSONObject("payload")?.optJSONObject("value")

                        )
                    }
                    payloadJsonObj?.optJSONObject("value")?.has("aiid") == true -> {
                        SpecActionParam.parseFromSmarthomeScene(
                            json.optJSONObject("payload")?.optJSONObject("value")
                        )
                    }
                    else -> {
                        payloadJsonObj?.opt("value")
                    }
                }
            } else payloadJsonObj?.opt("value")
            val pType = if (json?.has("protocol_type") == true) {
                json.optInt("protocol_type")
            } else {
                2
            }
            val sortKey = json?.optInt("rank")?:ConstantType.DEFAULT_TPL_RANK
//            val saId =
//                json?.optInt("sa_id")?.let { if (it == 0) json.optInt("related_sa_id") else it }
//                    ?: 0
//            if(saId == 0){
//                MiJiaLog.onlyLogcat("jmd236sss","saId == 0 did=${did} std_sa_id=${json?.optString("std_sa_id")}  std_rule_id=${json?.optString("std_rule_id")}")
//            }
            return ATplItem(
                did,
                hid,
                rid,
                json?.optString("name") ?: "",
                json?.optInt("sa_id") ?: 0,
                json?.optString("std_sa_id"),
                json?.optInt("std_rule_id")?:0,
                json?.optInt("related_sa_id")?:0,
                json?.optJSONObject("groupInfo")?.optInt("id") ?: -1,
                json?.optJSONObject("groupInfo")?.optString("intro"),
                payloadJsonObj?.optString("command"),
                value,
                payloadJsonObj?.optString("plug_id"),
                json?.optInt("tr_id"),
                optAttr(payloadJsonObj?.optJSONObject("attr"), payloadJsonObj?.optJSONObject("attr_new")),
                json?.optJSONObject("shadow_action")?.let { ShadowTCAItem.parseFromJson(it) },
                isSpec, pType,from,
                sortKey
            )

        }

        private fun isSpec(value: Any?): Boolean {
            if (value == null) {
                return false
            }
            if (value is JSONArray) {//property的情况
                if(value.length()>0){
                    val tmpValue:JSONObject? = value.optJSONObject(0)
                    if ( tmpValue!=null && (tmpValue.has("piid") || tmpValue.has("aiid") || tmpValue.has("eiid"))) {
                        //TODO: by jiangmeidan 需要确认一下，是否可以覆盖到所有的情况
                        return true
                    }
                }

            }
            if (value is JSONObject && value.has("aiid") && value.has("siid")) {
                return true
            }
            return false
        }

        private fun optAttr(attrJson: JSONObject?, attrNewJson: JSONObject?): CommonSceneOnline.SceneAttr? {
            val id: Int = attrJson?.optInt("attr_id") ?: attrNewJson?.optInt("attr_id") ?: 0
            val params: JSONObject? =
                attrJson?.optJSONObject("params") ?: attrNewJson?.optJSONObject("params")
            if (id <= 0) return null
            if (id == 2001) {
                val numPicker = CommonSceneOnline.SceneAttrNumberPicker()
                numPicker.attrId = id
                numPicker.maxValue = params?.optDouble("max_val")?.toFloat() ?: 0f
                numPicker.minValue = params?.optDouble("min_val")?.toFloat() ?: 0f
                numPicker.interval = params?.optDouble("interval")?.toFloat() ?: 0f

                val degreeFromConfig = params?.optString("degree")
                var degreeConfig:String? = ""
                if (!TextUtils.isEmpty(degreeFromConfig)) degreeConfig =
                    SceneManagerV2.getInstance().getDegreeConfig(degreeFromConfig)
                if (!TextUtils.isEmpty(degreeConfig)) numPicker.degree = degreeConfig
                else numPicker.degree = degreeFromConfig

                numPicker.jsonTag = params?.optString("json_val_tag")
                numPicker.defaultValue = params?.optDouble("default_val")?.toFloat() ?: 0f
                numPicker.subTitle = params?.optString("display_sub_title")
                val showTags =
                    attrJson?.optJSONArray("show_tags")
                        ?: attrNewJson?.optJSONArray("show_tags")

                for (i in 0 until (showTags?.length() ?: 0)) {
                    val tag = CommonSceneOnline.NumberPickerTag()
                    tag.from = showTags?.optJSONObject(i)?.optDouble("from")?.toFloat() ?: 0f
                    tag.to = showTags?.optJSONObject(i)?.optDouble("to")?.toFloat() ?: 0f
                    tag.tag = showTags?.optJSONObject(i)?.optString("tag") ?: ""
                    numPicker.showTags.add(tag)
                }
                return numPicker
            }
            return null
        }

    }
    override fun getKeyWord(
        specItem: Spec.SpecItem,
        param: TCAMatchParamSpecV2?,
        instance: SpecDevice?
    ): String {
        val keywordFromService = super.getKeyWord(specItem, param,instance)
        if(param == null || keywordFromService!="other") return keywordFromService
        val toggleKeyWord = MultiSwitchTitleHelper.getTogglePropertyKeyword(specItem, param)
        if (toggleKeyWord != "other") return toggleKeyWord
        return keywordFromService

    }
    override fun getMatchParam(model: String): TCAMatchParamSpecV2? {
        MiJiaLog.onlyLogcat("jmd236", "$mName   command::== $mCommand ")
        return MultiSwitchTitleHelper.getActionMatchParam(mCommand,mValue)

    }

}
===========================================================================



























全部状态测试打开的action筛选出来的设备比如：
[{"home":{"room_id":"244001726818","room_name":"漫游","home_name":"2326337897","home_id":"244001723220"},"device":{"token":"792ddb095b9fcff27164707251db7b76","isBlePropValid":true,"latitude":"0.0","comFlag":1153,"parentModel":"","longitude":"0.0","name":"床头灯","parentId":"","isOnline":true,"model":"philips.light.moonlight","ip":"192.168.31.198","extra":{"showGroupMember":false,"split":{},"mcu_version":"0038","fw_version":"2.0.9","isSubGroup":false,"pincodeType":0,"isSetPincode":0},"isOwner":false,"iconURL":"https://cdn.cnbj1.fds.api.mi-img.com/iotweb-user-center/developer_1729768168121nb41jDw0.png?GalaxyAccessKeyId=AKVGLQWBOVIRQ3XLEW&Expires=9223372036854775807&Signature=JP9Zhu9WiKMKl4sWZ7eS4L9GRB0=","version":"2.0.9","did":"99291562"},"instance_param":{"protocol_type":2,"std_sa_id":"147#philips.light.moonlight#1#prop.2.2","std_rule_id":147,"name":"亮度调节为","payload":{"device_name":"床头灯","did":"99291562","attr_new":{"params":{"json_val_tag":"equal","degree":"%","display_sub_title":"亮度调节为","interval":1,"min_val":1,"default_val":50,"max_val":100},"attr_id":2001},"value":[{"value":50,"siid":2,"piid":2}],"model":"philips.light.moonlight","command":"set_properties"}}}]

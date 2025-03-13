<template>
  <div class="weather" v-if="weatherData.city && weatherData.type">
    <span>{{ weatherData.city }}&nbsp;</span>
    <span>{{ weatherData.type }}&nbsp;</span>
    <span>{{ weatherData.temperature }}℃</span>
    <span class="sm-hidden">
      &nbsp;{{ weatherData.fengxiang }}&nbsp;
    </span>
    <span class="sm-hidden">{{ weatherData.fengli.split('-')[0] }}&nbsp;级</span>
  </div>
  <div class="weather" v-else>
    <span>天气数据获取失败</span>
  </div>
</template>

<script setup>
import { getAdcode, getAmapWeather, getVvhanWeather } from "@/api";
import { Error } from "@icon-park/vue-next";

// 高德开发者 Key
const mainKey = import.meta.env.VITE_WEATHER_KEY;

// 天气数据
const weatherData = reactive({
  city: null, // 城市
  type: null, // 天气现象
  temperature: null, // 实时气温
  fengxiang: null, // 风向描述
  fengli: null, // 风力级别
});

// 取出天气平均值
const getTemperature = (min, max) => {
  try {
    // 计算平均值并四舍五入
    const average = (Number(min) + Number(max)) / 2;
    return Math.round(average);
  } catch (error) {
    console.error("计算温度出现错误：", error);
    return "NaN";
  }
};

// 获取天气数据
const getWeatherData = async () => {
  try {
    if (!mainKey) {
      console.log("未配置高德 Key，使用备用天气接口");
      const result = await getVvhanWeather();
      console.log(result);
      const data = result.data;
      weatherData.city = data.city || "未知地区";
      weatherData.type = data.type;
      weatherData.temperature = getTemperature(
        data.low.replace("°C", ""),
        data.high.replace("°C", "")
      );
      weatherData.fengxiang = data.fengxiang;
      weatherData.fengli = data.fengli;
    } else {
      // 获取 Adcode
      const adCode = await getAdcode(mainKey);
      console.log(adCode);
      if (adCode.infocode !== "10000") {
        throw "地区查询失败";
      }
      weatherData.city = adCode.city;
      // 获取天气信息
      const result = await getAmapWeather(mainKey, adCode.adcode);
      weatherData.type = result.lives[0].weather;
      weatherData.temperature = result.lives[0].temperature;
      weatherData.fengxiang = result.lives[0].winddirection;
      weatherData.fengli = result.lives[0].windpower;
    }
  } catch (error) {
    console.error("天气信息获取失败:" + error);
    onError("天气信息获取失败");
  }
};

// 报错信息
const onError = (message) => {
  ElMessage({
    message,
    icon: h(Error, {
      theme: "filled",
      fill: "#efefef",
    }),
  });
  console.error(message);
};

onMounted(() => {
  // 调用获取天气
  getWeatherData();
});
</script>
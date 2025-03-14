<template>
  <div class="weather" v-if="weatherData.adCode.city && weatherData.weather.weather">
    <span>{{ weatherData.adCode.city }}&nbsp;</span>
    <span>{{ weatherData.weather.weather }}&nbsp;</span>
    <span>{{ weatherData.weather.temperature }}℃</span>
    <span class="sm-hidden">
      &nbsp;{{
        weatherData.weather.winddirection?.endsWith("风")
          ? weatherData.weather.winddirection
          : weatherData.weather.winddirection + "风"
      }}&nbsp;
    </span>
    <span class="sm-hidden">{{ weatherData.weather.windpower }}&nbsp;级</span>
  </div>
  <div class="weather" v-else>
    <span>天气数据获取失败</span>
  </div>
</template>

<script setup>
import { reactive, onMounted, h } from "vue";
import { ElMessage } from "element-plus";
import { getAdcode, getWeather, getOtherWeather } from "@/api";
import { Error } from "@icon-park/vue-next";

// 高德开发者 Key（请确保在环境变量中配置此 Key，否则将使用备用天气接口）
const mainKey = import.meta.env.VITE_WEATHER_KEY;

// 天气数据
const weatherData = reactive({
  adCode: {
    city: null, // 城市名称
    adcode: null, // 城市编码
  },
  weather: {
    weather: null,      // 天气现象
    temperature: null,  // 平均温度
    winddirection: null,// 风向
    windpower: null,    // 风力级别
  },
});

// 计算温度平均值方法（去掉温度字符串中的 "°C"）
const getTemperature = (low, high) => {
  try {
    const lowNum = Number(low.replace("°C", "").trim());
    const highNum = Number(high.replace("°C", "").trim());
    const average = (lowNum + highNum) / 2;
    return Math.round(average);
  } catch (error) {
    console.error("计算温度出现错误：", error);
    return "NaN";
  }
};

// 获取天气数据方法
const getWeatherData = async () => {
  try {
    // 如果未配置高德 API Key，则使用备用天气接口
    if (!mainKey) {
      console.log("未配置高德 API Key，使用备用天气接口");
      // 可根据实际情况传递参数，如 { city: "北京" }，此处采用自动识别（根据 IP）
      const result = await getOtherWeather();
      console.log("备用天气接口返回数据:", result);
      
      if (result) {
        // 根据接口返回数据赋值
        // 返回示例中：result.city 为城市名称，result.data 内包含天气详情
        const d = result.data;
        weatherData.adCode = {
          city: result.city || "未知地区",
          adcode: null,
        };
        weatherData.weather = {
          weather: d.type,
          temperature: getTemperature(d.low, d.high),
          winddirection: d.fengxiang,
          windpower: d.fengli,
        };
        console.log("最终天气数据：", weatherData);
      } else {
        throw new Error("备用天气接口返回数据错误");
      }
    } else {
      // 使用高德 API 获取地理位置信息
      const adCode = await getAdcode(mainKey);
      console.log("高德地理位置信息:", adCode);
      if (adCode.infocode !== "10000") {
        throw new Error("地区查询失败");
      }
      weatherData.adCode = {
        city: adCode.city,
        adcode: adCode.adcode,
      };
      // 根据 adcode 获取天气信息
      const result = await getWeather(mainKey, weatherData.adCode.adcode);
      console.log("高德天气信息:", result);
      if (result.lives && result.lives.length > 0) {
        weatherData.weather = {
          weather: result.lives[0].weather,
          temperature: result.lives[0].temperature,
          winddirection: result.lives[0].winddirection,
          windpower: result.lives[0].windpower,
        };
      } else {
        throw new Error("未获取到高德天气信息");
      }
    }
  } catch (error) {
    console.error("天气信息获取失败:", error);
    onError("天气信息获取失败");
  }
};

// 错误提示方法
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
  // 调用获取天气数据
  getWeatherData();
});
</script>
<template>
  <div class="weather" v-if="isWeatherDataValid">
    <span>{{ weatherData.adCode.city }}&nbsp;</span>
    <span>{{ weatherData.weather.weather }}&nbsp;</span>
    <span>{{ weatherData.weather.temperature }}℃</span>
    <span class="sm-hidden">
      &nbsp;{{ weatherData.weather.winddirection + '风' }}&nbsp;
    </span>
    <span class="sm-hidden">{{ weatherData.weather.windpower }}&nbsp;级</span>
  </div>
  <div class="weather" v-else>
    <span>天气数据获取失败</span>
  </div>
</template>

<script setup>
import { getAdcode, getWeather, getOtherWeather } from "@/api";
import { Error } from "@icon-park/vue-next";

// 高德开发者 Key
const mainKey = import.meta.env.VITE_WEATHER_KEY;

// 天气数据
const weatherData = reactive({
  adCode: {
    city: null, // 城市
    adcode: null, // 城市编码
  },
  weather: {
    weather: null, // 天气现象
    temperature: null, // 实时气温
    winddirection: null, // 风向描述
    windpower: null, // 风力级别
  },
});

// 检查天气数据是否有效
const isWeatherDataValid = computed(() => {
  return (
    weatherData.adCode.city &&
    weatherData.weather.weather &&
    weatherData.weather.temperature !== null
  );
});

// 获取天气数据
const getWeatherData = async () => {
  try {
    if (!mainKey) {
      console.log("未配置高德 API，使用备用天气接口");
      const result = await getOtherWeather(); // 自动识别位置
      if (result && result.success && result.data) {
        const data = result.data;

        weatherData.adCode = {
          city: result.city || "未知地区", // 城市名称
        };

        weatherData.weather = {
          weather: data.type, // 天气现象
          temperature: getTemperature(data.low.replace("°C", ""), data.high.replace("°C", "")), // 温度
          winddirection: data.fengxiang, // 风向
          windpower: data.fengli.replace("级", ""), // 风力级别
        };

        console.log("weatherData:", weatherData);
      } else {
        throw new Error("备用天气接口返回数据格式错误或数据获取失败");
      }
    } else {
      // 使用高德 API 获取天气数据（保持不变）
      const adCode = await getAdcode(mainKey);
      if (adCode.infocode !== "10000") {
        throw new Error("地区查询失败");
      }
      weatherData.adCode = {
        city: adCode.city,
        adcode: adCode.adcode,
      };
      const result = await getWeather(mainKey, weatherData.adCode.adcode);
      weatherData.weather = {
        weather: result.lives[0].weather,
        temperature: result.lives[0].temperature,
        winddirection: result.lives[0].winddirection,
        windpower: result.lives[0].windpower,
      };
    }
  } catch (error) {
    console.error("天气信息获取失败:", error);
    onError("天气信息获取失败");
  }
};

// 计算平均温度
const getTemperature = (min, max) => {
  const average = (Number(min) + Number(max)) / 2;
  return Math.round(average);
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
  getWeatherData();
});
</script>
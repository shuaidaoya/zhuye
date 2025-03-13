<template>
  <div class="weather">
    <span v-if="weatherData.loading">加载中...</span>
    <template v-else>
      <span v-if="weatherData.city">{{ weatherData.city }}&nbsp;</span>
      <span v-if="weatherData.type">{{ weatherData.type }}&nbsp;</span>
      <span v-if="weatherData.temperature">{{ weatherData.temperature }}</span>
      <span class="sm-hidden" v-if="weatherData.fengxiang">
        &nbsp;{{ weatherData.fengxiang }}&nbsp;
      </span>
      <span class="sm-hidden" v-if="weatherData.fengli">
        {{ weatherData.fengli.split('-')[0] }}&nbsp;级
      </span>
      <span v-if="!weatherData.city || !weatherData.type">天气数据获取失败</span>
    </template>
  </div>
</template>

<script setup>
import { getAdcode, getAmapWeather, getVvhanWeather } from "@/api";
import { Error } from "@icon-park/vue-next";
import { reactive, onMounted, h } from "vue";
import { ElMessage } from "element-plus";

const mainKey = import.meta.env.VITE_WEATHER_KEY;

const weatherData = reactive({
  loading: true,
  city: null,
  type: null,
  temperature: null,
  fengxiang: null,
  fengli: null,
});

const getTemperature = (min, max) => {
  const numMin = parseFloat(min?.replace(/[^0-9.-]/g, ''));
  const numMax = parseFloat(max?.replace(/[^0-9.-]/g, ''));
  
  if (isNaN(numMin)) return numMax !== undefined ? `${numMax}℃` : 'N/A';
  if (isNaN(numMax)) return numMin !== undefined ? `${numMin}℃` : 'N/A';
  
  return `${Math.round((numMin + numMax) / 2)}℃`;
};

const getWeatherData = async () => {
  try {
    if (!mainKey) {
      console.log("使用韩小韩备用接口");
      const result = await getVvhanWeather();
      if (result?.success) {
        // 修正城市字段路径
        weatherData.city = result.city || "未知地区";
        weatherData.type = result.data.type;
        weatherData.temperature = getTemperature(
          result.data.low.replace("°C", ""),
          result.data.high.replace("°C", "")
        );
        weatherData.fengxiang = result.data.fengxiang;
        weatherData.fengli = result.data.fengli;
      } else {
        throw new Error(result?.message || "天气数据异常");
      }
    } else {
      // 完整的高德处理逻辑
      const adCode = await getAdcode(mainKey);
      console.log("高德地理位置响应：", adCode);
      
      if (adCode.infocode !== "10000") {
        throw new Error(`[${adCode.infocode}] ${adCode.info}`);
      }
      
      weatherData.city = adCode.city;
      const weatherRes = await getAmapWeather(mainKey, adCode.adcode);
      console.log("高德天气响应：", weatherRes);
      
      if (weatherRes?.lives?.[0]) {
        const liveData = weatherRes.lives[0];
        weatherData.type = liveData.weather;
        weatherData.temperature = liveData.temperature;
        weatherData.fengxiang = liveData.winddirection;
        weatherData.fengli = liveData.windpower;
      } else {
        throw new Error(weatherRes?.info || "天气数据解析失败");
      }
    }
  } catch (error) {
    console.error("天气获取失败：", error);
    onError(error.message);
  } finally {
    weatherData.loading = false;
  }
};

const onError = (message) => {
  ElMessage({
    message: `天气错误: ${message}`,
    icon: h(Error, { theme: "filled", fill: "#efefef" }),
  });
};

onMounted(() => {
  getWeatherData();
});
</script>
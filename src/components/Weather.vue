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
import { getAdcode, getWeather, getRegeo, getOtherIp } from "@/api";
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

// 获取天气数据
const getWeatherData = async () => {
  try {
    // 验证 Key 是否存在
    if (!mainKey) {
      throw "未配置高德地图 Key";
    }

    let adcode = null;
    let city = null;

    // 1. 优先尝试浏览器地理位置定位（用户授权）
    try {
      console.log("尝试使用浏览器定位...");
      const position = await new Promise((resolve, reject) => {
        navigator.geolocation.getCurrentPosition(resolve, reject, {
          timeout: 5000,
        });
      });
      const location = `${position.coords.longitude},${position.coords.latitude}`;
      const regeoResult = await getRegeo(mainKey, location);
      if (regeoResult.infocode === "10000") {
        adcode = regeoResult.regeocode.addressComponent.adcode;
        city = regeoResult.regeocode.addressComponent.city || regeoResult.regeocode.addressComponent.province;
        console.log("浏览器定位成功:", city);
      }
    } catch (error) {
      console.warn("浏览器定位失败或用户拒绝授权:", error);

      // 2. 浏览器定位失败，尝试高德 IP 定位
      try {
        console.log("尝试使用高德 IP 定位...");
        const ipResult = await getAdcode(mainKey);
        if (
          ipResult.infocode === "10000" &&
          ipResult.adcode &&
          typeof ipResult.adcode === "string"
        ) {
          adcode = ipResult.adcode;
          city = ipResult.city;
          console.log("高德 IP 定位成功:", city);
        }
      } catch (error) {
        console.warn("高德 IP 定位失败:", error);
      }

      // 3. 高德 IP 定位也失败，尝试第三方 IP 定位（无感）
      if (!adcode) {
        try {
          console.log("尝试使用第三方 IP 定位...");
          const otherIpResult = await getOtherIp();
          if (otherIpResult.success) {
            const location = `${otherIpResult.longitude},${otherIpResult.latitude}`;
            const regeoResult = await getRegeo(mainKey, location);
            if (regeoResult.infocode === "10000") {
              adcode = regeoResult.regeocode.addressComponent.adcode;
              city = regeoResult.regeocode.addressComponent.city || regeoResult.regeocode.addressComponent.province;
              console.log("第三方 IP 定位成功:", city);
            }
          }
        } catch (error) {
          console.warn("第三方 IP 定位失败:", error);
        }
      }
    }

    // 4. 如果所有定位都失败，降级到默认城市（北京: 110000）
    if (!adcode) {
      console.warn("定位全部失败，使用默认城市（北京）");
      adcode = "110000";
      city = "北京市";
    }

    weatherData.adCode = {
      city: city,
      adcode: adcode,
    };

    // 获取天气信息
    const result = await getWeather(mainKey, weatherData.adCode.adcode);
    if (!result.lives || result.lives.length === 0) {
      throw "天气数据为空";
    }
    weatherData.weather = {
      weather: result.lives[0].weather,
      temperature: result.lives[0].temperature,
      winddirection: result.lives[0].winddirection,
      windpower: result.lives[0].windpower,
    };
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

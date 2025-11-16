<template>
  <div class="min-h-screen flex flex-col items-center bg-gray-50 py-10 px-4">
    <h2 class="text-2xl font-semibold text-gray-800 mb-8">ĐĂNG KÝ SẢN PHẨM</h2>

    <!-- Toast -->
    <transition name="slide-fade">
      <div
        v-if="toast.show"
        :class="[
          'fixed top-5 right-5 min-w-[250px] px-4 py-3 rounded-lg shadow-md text-gray-800 bg-white border-l-4 z-50',
          toast.type === 'success' ? 'border-green-500' : 'border-red-500',
        ]"
      >
        {{ toast.message }}
      </div>
    </transition>

    <div v-if="loading" class="text-center text-gray-500 py-6">Đang tải dữ liệu...</div>

    <form v-else @submit.prevent="submitProduct" class="w-full max-w-3xl space-y-5">
      <div class="form-row">
        <label>Tên sản phẩm</label>
        <div class="flex-1">
          <input
            v-model="product.name"
            class="input"
            :class="{ error: errors.name }"
            placeholder="Nhập tên sản phẩm"
          />
          <p v-if="errors.name" class="error-msg">{{ errors.name }}</p>
        </div>
      </div>

      <div class="form-row">
        <label>Danh mục</label>
        <div class="flex-1">
          <select
            v-model="product.category"
            class="input"
            :class="{ error: errors.category }"
          >
            <option disabled value="">-- Chọn danh mục --</option>
            <option v-for="cat in categories" :key="cat.id" :value="String(cat.madm)">
              {{ cat.tendm }}
            </option>
          </select>
          <p v-if="errors.category" class="error-msg">{{ errors.category }}</p>
        </div>
      </div>

      <div class="form-row">
        <label>Tình trạng sản phẩm</label>
        <div class="flex-1">
          <input
            v-model="product.condition"
            class="input"
            :class="{ error: errors.condition }"
            placeholder="Ví dụ: Mới, Cũ, Hàng Like New..."
          />
          <p v-if="errors.condition" class="error-msg">{{ errors.condition }}</p>
        </div>
      </div>

      <div class="form-row">
        <label>Thành phố</label>
        <div class="flex-1">
          <select v-model="product.city" class="input" :class="{ error: errors.city }">
            <option disabled value="">-- Chọn thành phố --</option>
            <option v-for="c in cities" :key="c.matp" :value="String(c.matp)">
              {{ c.tentp }}
            </option>
          </select>
          <p v-if="errors.city" class="error-msg">{{ errors.city }}</p>
        </div>
      </div>

      <div class="form-row">
        <label>Ảnh sản phẩm</label>
        <div class="flex gap-4 flex-wrap">
          <div
            v-for="(img, index) in product.images"
            :key="index"
            class="w-32 h-32 border border-gray-300 rounded-md flex items-center justify-center bg-gray-100 relative cursor-pointer"
            @click="removeImage(index)"
          >
            <img
              v-if="img.preview"
              :src="img.preview"
              class="object-cover w-full h-full rounded-md"
            />
            <span v-else class="text-gray-400 text-center">Ảnh</span>
            <button
              v-if="img.preview"
              type="button"
              @click.stop="removeImage(index)"
              class="absolute top-1 right-1 text-red-500 bg-white rounded-full w-5 h-5 flex items-center justify-center text-sm"
            >
              ✕
            </button>
          </div>
          <label
            v-if="product.images.length < 3"
            class="w-32 h-32 border border-dashed border-gray-400 rounded-md flex items-center justify-center cursor-pointer text-gray-400"
          >
            Thêm ảnh
            <input
              type="file"
              class="hidden"
              accept="image/png, image/jpeg, image/jpg"
              @change="handleImage"
            />
          </label>
        </div>
        <p v-if="errors.images" class="error-msg">{{ errors.images }}</p>
      </div>

      <div class="pt-6 flex justify-end gap-3">
        <button type="submit" class="btn-primary">Đăng ký sản phẩm</button>
      </div>
    </form>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import axios from "axios";
import Cookies from "js-cookie";

const API_URL = "https://daugiabe-production.up.railway.app"; // dùng URL cố định cho BE

const product = ref({
  name: "",
  category: "",
  condition: "",
  city: "",
  images: [],
});

const categories = ref([]);
const cities = ref([]);
const errors = ref({});
const loading = ref(true);
const toast = ref({ show: false, message: "", type: "success" });

const showToast = (msg, type = "success") => {
  toast.value = { show: true, message: msg, type };
  setTimeout(() => (toast.value.show = false), 3000);
};

// Xử lý ảnh: chỉ preview; không rename trên FE (để BE sanitize/đổi tên)
const handleImage = (e) => {
  const file = e.target.files[0];
  if (!file) return;

  const validTypes = ["image/png", "image/jpeg", "image/jpg"];
  if (!validTypes.includes(file.type)) {
    showToast("Chỉ chấp nhận ảnh PNG, JPG, JPEG!", "error");
    return;
  }

  if (product.value.images.length >= 3) return;

  const reader = new FileReader();
  reader.onload = () => {
    product.value.images.push({
      file,
      preview: reader.result,
    });
  };
  reader.readAsDataURL(file);
  e.target.value = "";
};

const removeImage = (index) => {
  product.value.images.splice(index, 1);
};

// Submit sản phẩm và upload ảnh
const submitProduct = async () => {
  errors.value = {};
  if (!product.value.name.trim()) errors.value.name = "Nhập tên sản phẩm.";
  if (!product.value.category) errors.value.category = "Chọn danh mục.";
  if (!product.value.condition.trim())
    errors.value.condition = "Nhập tình trạng sản phẩm.";
  if (!product.value.city) errors.value.city = "Chọn thành phố.";
  if (product.value.images.length === 0) errors.value.images = "Thêm ít nhất 1 ảnh.";

  if (Object.keys(errors.value).length > 0) return;

  const token = Cookies.get("jwt_token");
  if (!token) {
    showToast("Phiên đăng nhập đã hết hạn!", "error");
    return;
  }

  try {
    // 1) Tạo sản phẩm
    const productPayload = {
      madm: product.value.category,
      matp: product.value.city,
      tensp: product.value.name,
      tinhtrangsp: product.value.condition,
    };
    const res = await axios.post(
      `${API_URL}/api/products/create`,
      productPayload,
      { headers: { Authorization: `Bearer ${token}` } }
    );

    if (res.data?.code !== 200) {
      showToast(res.data?.message || "Tạo sản phẩm thất bại!", "error");
      return;
    }

    const masp = res.data.result.masp;
    showToast("Tạo sản phẩm thành công!", "success");

    // 2) Upload ảnh (Multipart) — để BE xử lý tên file
    if (product.value.images.length > 0) {
      const formData = new FormData();
      formData.append("masp", masp);
      product.value.images.forEach((img) => formData.append("files", img.file));

      const imageRes = await axios.post(
        `${API_URL}/api/images/upload`,
        formData,
        {
          headers: {
            Authorization: `Bearer ${token}`,
            "Content-Type": "multipart/form-data",
          },
        }
      );

      if (imageRes.data?.code === 200) showToast("Upload ảnh thành công!", "success");
      else showToast(imageRes.data?.message || "Upload ảnh thất bại!", "error");
    }

    // 3) Reset form
    product.value = { name: "", category: "", condition: "", city: "", images: [] };
  } catch (err) {
    console.error(err);
    showToast("Đăng ký sản phẩm thất bại!", "error");
    
  }
};

// Fetch categories & cities
onMounted(async () => {
  try {
    const [catRes, cityRes] = await Promise.all([
      axios.get(`${API_URL}/api/cates/find-all`),
      axios.get(`${API_URL}/api/cities/find-all`),
    ]);

    if (catRes.data?.result) categories.value = catRes.data.result;
    if (cityRes.data?.result) cities.value = cityRes.data.result;
  } catch (err) {
    console.error(err);
    showToast("Không thể tải dữ liệu danh mục hoặc thành phố!", "error");
  } finally {
    loading.value = false;
  }
});
</script>

<style scoped>
@import "@/assets/styles/auth.css";
@import "@/assets/styles/toast.css";
.input { border-radius: 0.5rem; }
.input.error { border-color: red; }

.error-msg {
  color: red;
  font-size: 0.875rem;
  margin-top: 0.25rem;
}

.btn-primary {
  @apply w-full py-2.5 rounded-lg font-semibold text-white shadow-md transition;
  background-color: #22b8cf;
}
.btn-primary:hover {
  filter: brightness(1.05);
  background-color: #1098ad;
}
</style>
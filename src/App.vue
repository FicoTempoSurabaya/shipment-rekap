<script setup>
import { ref, computed, onMounted } from 'vue';
import { Truck, Save, Trash2, Calendar, Search, Loader2 } from 'lucide-vue-next';
import Swal from 'sweetalert2';

// --- CONFIGURATION ---
const API_URL = 'https://script.google.com/macros/s/AKfycbze8N3rocRoWDzAwtJoJ9uQkRjlxLgxxLtSORV9upHhR2e7WcXWj1LbQfyKwAdw1_ahmQ/exec'; 

// --- STATE ---
const users = ref([]);
const holidays = ref({}); 
const selectedUser = ref(null); 
const dateStart = ref('');
const dateEnd = ref('');
const shipmentsData = ref({}); 
const isLoading = ref(false);
const isSaving = ref(false);
const showTable = ref(false);

// --- FETCH INITIAL DATA ---
const fetchInitialData = async () => {
  try {
    const res = await fetch(`${API_URL}?action=getInitialData`);
    const data = await res.json();
    users.value = data.users;
    holidays.value = data.holidays; 
  } catch (error) {
    Swal.fire('Error', 'Gagal memuat data awal.', 'error');
  }
};

onMounted(() => {
  fetchInitialData();
});

// --- LOGIC: DATE GENERATOR ---
const generatedDates = computed(() => {
  if (!dateStart.value || !dateEnd.value) return [];
  
  const dates = [];
  let current = new Date(dateStart.value);
  const end = new Date(dateEnd.value);
  
  while (current <= end) {
    const yyyy = current.getFullYear();
    const mm = String(current.getMonth() + 1).padStart(2, '0');
    const dd = String(current.getDate()).padStart(2, '0');
    const strDate = `${yyyy}-${mm}-${dd}`;
    
    // Cek Hari Minggu
    const isSunday = current.getDay() === 0;
    
    // Cek Hari Libur dari Database
    const holidayDesc = holidays.value[strDate]; 
    const isHolidayDB = !!holidayDesc; 
    
    // Logika Penentuan Label Teks
    let labelHari = 'Kerja';
    if (isHolidayDB) {
      labelHari = holidayDesc; 
    } else if (isSunday) {
      labelHari = 'Minggu';
    }

    dates.push({
      fullDate: strDate,
      displayDate: new Intl.DateTimeFormat('id-ID', { day: 'numeric', month: 'short', weekday: 'short' }).format(current),
      isLocked: isSunday || isHolidayDB, 
      dayType: labelHari 
    });
    
    current.setDate(current.getDate() + 1);
  }
  return dates;
});

// --- LOGIC: CARDBOARD STATS ---
const stats = computed(() => {
  const dates = generatedDates.value;
  if (dates.length === 0) return { hariKerja: 0, hariEfektif: 0 };

  const hariKerja = dates.filter(d => !d.isLocked).length;
  const hariEfektif = dates.filter(d => {
    const val = shipmentsData.value[d.fullDate];
    return !d.isLocked && val && val.length > 0;
  }).length;

  return { hariKerja, hariEfektif };
});

// --- LOGIC: DYNAMIC TRAFFIC LIGHT COLOR ---
const getEffectiveCardClass = computed(() => {
  const { hariKerja, hariEfektif } = stats.value;

  // 1. Jika Hari Kerja 0 (belum pilih tanggal), default style
  if (hariKerja === 0) return 'bg-slate-800 text-slate-400';

  // 2. Jika Hari Efektif 0 (Belum ada isian sama sekali) -> HITAM
  if (hariEfektif === 0) {
    return 'bg-gradient-to-r from-slate-500 to-slate-800 text-white shadow-inner border border-slate-700'; 
  }

  // 3. Jika SAMA PERSIS (Selesai/Perfect) -> SAMA DENGAN HARI KERJA (Indigo Gradient)
  if (hariEfektif === hariKerja) {
    return 'bg-gradient-to-bl from-indigo-600 to-indigo-800 text-white pop-primary';
  }

  // 4. Jika Variatif (0 < X < Hari Kerja) -> Traffic Light System
  const percentage = hariEfektif / hariKerja;
  
  if (percentage < 0.16) {
    // Merah (0% - 33%)
    return 'bg-gradient-to-br from-slate-500 to-slate-700 text-white shadow-sm shadow-slate-200 border-t border-slate-400';
  } else if (percentage < 0.26) {
    return 'bg-gradient-to-br from-pink-500 to-rose-600 text-white shadow-sm shadow-rose-200 border-t border-rose-400';
  } else if (percentage < 0.36) {
    return 'bg-gradient-to-br from-amber-200 to-yellow-500 text-white shadow-sm shadow-yellow-200 border-t border-yellow-400';
  } else if (percentage < 0.51) {
    return 'bg-gradient-to-br from-amber-500 to-amber-700 text-white shadow-sm shadow-amber-200 border-t border-amber-400';
  } else if (percentage < 0.66) {
    return 'bg-gradient-to-br from-amber-500 to-amber-700 text-white shadow-sm shadow-amber-200 border-t border-amber-400';
  } else if (percentage < 0.76) {
    return 'bg-gradient-to-br from-amber-500 to-amber-700 text-white shadow-sm shadow-amber-200 border-t border-amber-400';
  } else if (percentage < 0.86) {
    return 'bg-gradient-to-br from-yellow-600 to-emerald-500 text-white shadow-sm shadow-emerald-200 border-t border-emerald-400';
  } else {
    // Hijau/Emerald (75% - 99%)
    return 'bg-gradient-to-br from-emerald-500 to-indigo-500 text-white shadow-sm shadow-indigo-200 border-t border-indigo-400';
  }
});

// --- ACTIONS ---
const handleShow = async () => {
  if (!selectedUser.value || !dateStart.value || !dateEnd.value) {
    Swal.fire('Perhatian', 'Mohon lengkapi Nama dan Rentang Tanggal', 'warning');
    return;
  }
  if (dateEnd.value < dateStart.value) {
    Swal.fire('Error', 'Tanggal Selesai tidak boleh sebelum Tanggal Mulai', 'error');
    return;
  }

  isLoading.value = true;
  showTable.value = false;
  shipmentsData.value = {}; 

  try {
    const params = new URLSearchParams({
      action: 'getShipments',
      nama: selectedUser.value.nama,
      startDate: dateStart.value,
      endDate: dateEnd.value
    });
    
    const res = await fetch(`${API_URL}?${params.toString()}`);
    const data = await res.json();
    
    data.shipments.forEach(item => {
      shipmentsData.value[item.tanggal] = item.shipmentId;
    });
    
    showTable.value = true;
  } catch (err) {
    Swal.fire('Error', 'Gagal memuat data shipment', 'error');
  } finally {
    isLoading.value = false;
  }
};

const handleSaveRow = async (dateObj) => {
  const value = shipmentsData.value[dateObj.fullDate];
  
  if (!value || !/^\d{10}$/.test(value)) {
    Swal.fire('Invalid', 'Shipment ID wajib 10 digit angka.', 'warning');
    return;
  }

  isSaving.value = true;
  try {
    await fetch(API_URL, {
      method: 'POST',
      body: JSON.stringify({
        action: 'saveShipment',
        nik: selectedUser.value.nik,
        nama: selectedUser.value.nama,
        tanggal: dateObj.fullDate,
        shipmentId: value
      })
    });
    
    Swal.fire({
      icon: 'success',
      title: 'Berhasil',
      text: 'Data shipment berhasil disimpan',
      timer: 1500,
      showConfirmButton: false
    });
  } catch (err) {
    Swal.fire('Error', 'Gagal menyimpan data', 'error');
  } finally {
    isSaving.value = false;
  }
};

const handleDeleteRow = async (dateObj) => {
  const result = await Swal.fire({
    title: 'Hapus Data?',
    text: `Hapus shipment tanggal ${dateObj.displayDate}?`,
    icon: 'warning',
    showCancelButton: true,
    confirmButtonColor: '#ef4444',
    confirmButtonText: 'Ya, Hapus!'
  });

  if (result.isConfirmed) {
    isSaving.value = true;
    try {
      await fetch(API_URL, {
        method: 'POST',
        body: JSON.stringify({
          action: 'deleteShipment',
          nama: selectedUser.value.nama,
          tanggal: dateObj.fullDate
        })
      });

      delete shipmentsData.value[dateObj.fullDate];
      
      Swal.fire('Terhapus!', 'Data shipment telah dihapus.', 'success');
    } catch (err) {
      Swal.fire('Error', 'Gagal menghapus data', 'error');
    } finally {
      isSaving.value = false;
    }
  }
};
</script>

<template>
  <div class="min-h-screen bg-slate-100 text-slate-800 pb-20 font-sans selection:bg-indigo-100">
    
    <header class="bg-white/80 backdrop-blur-md border-b border-white/50 sticky top-0 z-30 pop-light">
      <div class="max-w-md mx-auto px-4 py-4 flex items-center space-x-3">
        <div class="bg-linear-to-br from-indigo-500 to-indigo-700 p-2 rounded-xl pop-primary">
          <Truck class="w-6 h-6 text-white drop-shadow-sm" />
        </div>
        <div>
          <h1 class="text-xs font-bold text-slate-400 uppercase tracking-wider text-shadow-sm">Selamat Datang</h1>
          <h2 class="text-lg font-black text-slate-800 leading-tight drop-shadow-sm">Shipment Rekap System</h2>
        </div>
      </div>
    </header>

    <main class="max-w-md mx-auto px-4 mt-6 space-y-6">
      
      <div class="bg-white rounded-2xl p-6 space-y-5 pop-light">
        <div>
          <label class="block text-sm font-bold text-slate-600 mb-2 ml-1">Nama Lengkap</label>
          <div class="relative">
            <select 
              v-model="selectedUser" 
              class="w-full appearance-none rounded-xl py-3 px-4 pr-8 text-slate-700 font-medium focus:outline-none focus:ring-2 focus:ring-indigo-500/50 transition-all duration-300 inset-3d"
            >
              <option :value="null" disabled>Pilih Nama Karyawan...</option>
              <option v-for="u in users" :key="u.nik" :value="u">
                {{ u.nama }}
              </option>
            </select>
            <div class="pointer-events-none absolute inset-y-0 right-0 flex items-center px-4 text-slate-500">
              <svg class="fill-current h-4 w-4 drop-shadow-sm" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20"><path d="M9.293 12.95l.707.707L15.657 8l-1.414-1.414L10 10.828 5.757 6.586 4.343 8z"/></svg>
            </div>
          </div>
        </div>

        <div class="grid grid-cols-2 gap-4">
          <div>
            <label class="block text-xs font-bold text-slate-500 mb-2 ml-1 uppercase">Mulai</label>
            <input type="date" v-model="dateStart" class="w-full rounded-xl py-2 px-3 text-slate-700 font-medium focus:ring-2 focus:ring-indigo-500/50 outline-none transition-all inset-3d" />
          </div>
          <div>
            <label class="block text-xs font-bold text-slate-500 mb-2 ml-1 uppercase">Selesai</label>
            <input type="date" v-model="dateEnd" class="w-full rounded-xl py-2 px-3 text-slate-700 font-medium focus:ring-2 focus:ring-indigo-500/50 outline-none transition-all inset-3d" />
          </div>
        </div>

        <button 
          @click="handleShow"
          :disabled="isLoading"
          class="w-full text-white font-bold py-3 px-4 rounded-xl transition-all duration-200 flex items-center justify-center space-x-2 pop-primary btn-press disabled:pointer-events-none"
          :class="isLoading 
            ? 'bg-linear-to-br from-slate-400 to-slate-500 cursor-not-allowed' 
            : 'bg-linear-to-br from-indigo-600 to-indigo-700 hover:from-indigo-500 hover:to-indigo-600'"
        >
          <Loader2 v-if="isLoading" class="animate-spin w-5 h-5 drop-shadow-sm" />
          <Search v-else class="w-5 h-5 drop-shadow-sm" />
          <span class="drop-shadow-sm">{{ isLoading ? 'Memuat Data...' : 'Tampilkan Data' }}</span>
        </button>
      </div>

      <transition
        enter-active-class="transition duration-500 ease-out"
        enter-from-class="transform translate-y-4 opacity-0"
        enter-to-class="transform translate-y-0 opacity-100"
        leave-active-class="transition duration-300 ease-in"
        leave-from-class="opacity-100"
        leave-to-class="opacity-0"
      >
        <div v-if="showTable" class="space-y-6">
          
          <div class="grid grid-cols-2 gap-4">
            
            <div class="bg-linear-to-br from-indigo-600 to-indigo-800 p-4 rounded-2xl flex flex-col items-center justify-center text-center text-white pop-primary relative overflow-hidden group">
              <div class="absolute top-0 right-0 -mt-2 -mr-2 w-16 h-16 bg-white opacity-10 rounded-full blur-xl group-hover:scale-150 transition-transform duration-700"></div>
              
              <span class="text-xs text-indigo-100 font-bold uppercase tracking-wider drop-shadow-sm relative z-10">Hari Kerja</span>
              <span class="text-3xl font-black mt-1 drop-shadow-md relative z-10">{{ stats.hariKerja }}</span>
            </div>

            <div 
              class="p-4 rounded-2xl flex flex-col items-center justify-center text-center transition-all duration-500 ease-in-out relative overflow-hidden"
              :class="getEffectiveCardClass"
            >
              <div v-if="stats.hariEfektif === stats.hariKerja && stats.hariKerja > 0" ></div>

              <span class="text-xs font-bold uppercase tracking-wider drop-shadow-sm relative z-10 opacity-90">Hari Efektif</span>
              <span class="text-3xl font-black mt-1 drop-shadow-md relative z-10">{{ stats.hariEfektif }}</span>
            </div>
            
          </div>

          <div class="bg-white rounded-2xl overflow-hidden pop-light">
            <div class="grid grid-cols-12 bg-slate-50/80 border-b border-slate-200/80 py-3 px-4 text-xs font-bold text-slate-500 uppercase tracking-wider backdrop-blur-sm">
              <div class="col-span-4 text-left">Tanggal</div>
              <div class="col-span-5 text-center">Shipment ID</div>
              <div class="col-span-3 text-center">Aksi</div>
            </div>

            <div class="divide-y divide-slate-100/50">
              <div 
                v-for="dateObj in generatedDates" 
                :key="dateObj.fullDate"
                class="grid grid-cols-12 items-center py-3 px-4 transition-colors hover:bg-indigo-100"
                :class="{'bg-red-100': dateObj.isLocked}"
              >
                <div class="col-span-4 flex flex-col justify-center">
                  <span class="text-sm font-bold text-slate-700 drop-shadow-sm" :class="{'text-red-600': dateObj.isLocked}">
                    {{ dateObj.displayDate.split(',')[0] }}
                  </span>
                  <span class="text-[10px] text-slate-400 font-medium">
                    {{ dateObj.displayDate.split(',')[1] }}
                  </span>
                  <span v-if="dateObj.isLocked" class="text-[9px] text-red-500 font-bold uppercase mt-0.5 drop-shadow-sm">
                    {{ dateObj.dayType }}
                  </span>
                </div>

                <div class="col-span-5 pr-2">
                  <div v-if="dateObj.isLocked" class="text-center text-slate-300 font-bold select-none text-xl inset-3d rounded-lg py-2 bg-red-100/20 border-transparent">
                    -
                  </div>
                  <input 
                    v-else
                    type="tel" 
                    maxlength="10"
                    placeholder="Isi 10 digit..."
                    v-model="shipmentsData[dateObj.fullDate]"
                    class="w-full rounded-lg py-2 px-2 text-center text-sm font-mono font-bold placeholder-slate-400 transition-all focus:bg-white focus:ring-2 focus:ring-indigo-500/50 outline-none inset-3d"
                  />
                </div>

                <div class="col-span-3 flex justify-end gap-2">
                  <template v-if="!dateObj.isLocked">
                    <button 
                      @click="handleSaveRow(dateObj)"
                      class="p-2 bg-emerald-100 text-emerald-600 rounded-lg hover:bg-emerald-200 transition-all border-b-2 border-emerald-200 hover:border-emerald-300 active:border-b-0 active:translate-y-0.5 shadow-sm"
                      title="Simpan"
                    >
                      <Save class="w-4 h-4 drop-shadow-sm" />
                    </button>
                    <button 
                      @click="handleDeleteRow(dateObj)"
                      class="p-2 bg-red-100 text-red-600 rounded-lg hover:bg-red-200 transition-all border-b-2 border-red-200 hover:border-red-300 active:border-b-0 active:translate-y-0.5 shadow-sm"
                      title="Hapus"
                    >
                      <Trash2 class="w-4 h-4 drop-shadow-sm" />
                    </button>
                  </template>
                </div>
              </div>
            </div>
            
            <div class="h-3 bg-slate-50/50 inset-3d border-t-0 rounded-b-2xl"></div>
          </div>
        </div>
      </transition>

    </main>
  </div>
</template>
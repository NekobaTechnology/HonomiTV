<template>
    <div class="route-container">
        <HeaderBar />
        <main>
            <Navigation />
            <div class="reservation-condition-edit-container-wrapper">
                <SPHeaderBar />
                <div class="reservation-condition-edit-container">
                    <Breadcrumbs :crumbs="[
                        { name: 'ホーム', path: '/' },
                        { name: '録画予約', path: '/reservations/' },
                        { name: 'キーワード自動予約', path: '/reservations/conditions' },
                        { name: isNew ? '新規作成' : '条件の編集', path: '', disabled: true },
                    ]" />
                    <div class="reservation-condition-edit">
                        <div class="reservation-condition-edit__header">
                            <h2 class="reservation-condition-edit__title">
                                <div v-ripple class="reservation-condition-edit__title-back" @click="$router.push('/reservations/conditions')">
                                    <Icon icon="fluent:chevron-left-12-filled" width="27px" />
                                </div>
                                <span class="reservation-condition-edit__title-text">
                                    {{ isNew ? 'キーワード自動予約を新規作成' : 'キーワード自動予約の編集' }}
                                </span>
                            </h2>
                            <div class="reservation-condition-edit__actions">
                                <v-btn variant="text" color="text" @click="$router.push('/reservations/conditions')">キャンセル</v-btn>
                                <v-btn color="secondary" variant="flat" :loading="isSaving" :disabled="isLoading" @click="save">
                                    <Icon icon="fluent:save-20-regular" width="20px" height="20px" class="mr-1" />
                                    {{ isNew ? '登録する' : '保存する' }}
                                </v-btn>
                            </div>
                        </div>

                        <div v-if="isLoading" class="reservation-condition-edit__loading">
                            <Icon icon="line-md:loading-twotone-loop" class="spin" width="32px" height="32px" />
                        </div>

                        <div v-if="!isLoading && prefillSourceTitle !== null" class="reservation-condition-edit__prefill">
                            <Icon icon="fluent:arrow-repeat-all-20-regular" width="20px" height="20px" class="reservation-condition-edit__prefill-icon" />
                            <div class="reservation-condition-edit__prefill-text">
                                「{{ prefillSourceTitle }}」から、同じシリーズの各話に一致するように
                                キーワード「{{ searchCondition.keyword }}」<template v-if="prefillChannelName !== null">・放送局「{{ prefillChannelName }}」</template>・番組名のみ検索・重複回避 (同じチャンネル) を設定しました。
                                右の「条件に一致する番組」を確認して、必要なら条件を調整してから登録してください。
                            </div>
                        </div>
                        <div v-if="!isLoading" class="reservation-condition-edit__body">
                            <!-- 左: 番組検索条件 -->
                            <div class="reservation-condition-edit__search">
                                <div class="reservation-condition-edit__section-title">
                                    <Icon icon="fluent:search-20-filled" width="20px" height="20px" class="mr-1" />
                                    番組の検索条件
                                </div>
                                <div class="reservation-condition-edit__section-description">
                                    ここで指定した条件に一致する番組が自動的に録画予約されます。
                                    「検索」を押すと、現在の番組表で条件に一致する番組を確認できます。
                                </div>
                                <ProgramSearchFilters
                                    v-model:condition="searchCondition"
                                    :channels="allChannels"
                                    :is-searching="isSearching"
                                    @search="previewPrograms" />
                            </div>

                            <!-- 右: 有効/無効・プレビュー・録画設定 -->
                            <div class="reservation-condition-edit__side">
                                <!-- 条件の有効/無効 -->
                                <div class="reservation-condition-edit__section">
                                    <div class="reservation-condition-edit__section-header">
                                        <div class="reservation-condition-edit__section-title">
                                            <Icon icon="fluent:toggle-right-20-regular" width="20px" height="20px" class="mr-1" />
                                            この自動予約を有効にする
                                        </div>
                                        <v-switch
                                            v-model="searchCondition.is_enabled"
                                            color="primary"
                                            density="compact"
                                            hide-details>
                                        </v-switch>
                                    </div>
                                    <div class="reservation-condition-edit__section-description mb-0">
                                        無効にすると、条件は残したまま新しい録画予約の自動追加だけを止められます。
                                    </div>
                                </div>

                                <!-- プレビュー -->
                                <div class="reservation-condition-edit__section">
                                    <div class="reservation-condition-edit__section-title">
                                        <Icon icon="fluent:tv-20-regular" width="20px" height="20px" class="mr-1" />
                                        条件に一致する番組
                                        <span v-if="hasPreviewed" class="reservation-condition-edit__section-count">{{ previewTotal }}件</span>
                                    </div>
                                    <div v-if="!hasPreviewed" class="reservation-condition-edit__section-description mb-0">
                                        左の「検索」を押すと、現在の番組表から条件に一致する番組を表示します。
                                        ここに意図しない番組が多く出る場合は、除外キーワードやチャンネルで絞り込んでください。
                                    </div>
                                    <div v-else-if="previewPrograms_.length === 0" class="reservation-condition-edit__section-description mb-0">
                                        現在の番組表には条件に一致する番組がありません。
                                        今後の番組表更新で一致する番組が見つかれば自動的に予約されます。
                                    </div>
                                    <div v-else class="reservation-condition-edit__preview">
                                        <div v-for="program in previewPrograms_" :key="program.id" class="reservation-condition-edit__preview-item">
                                            <div class="reservation-condition-edit__preview-time">
                                                {{ ProgramUtils.getProgramTime(program) }}
                                                <span class="reservation-condition-edit__preview-channel">{{ getChannelName(program) }}</span>
                                            </div>
                                            <div class="reservation-condition-edit__preview-title"
                                                v-html="ProgramUtils.decorateProgramInfo(program, 'title')"></div>
                                        </div>
                                        <div v-if="previewTotal > previewPrograms_.length" class="reservation-condition-edit__section-description mb-0 mt-2">
                                            ほか {{ previewTotal - previewPrograms_.length }} 件
                                        </div>
                                    </div>
                                </div>

                                <!-- 録画設定 -->
                                <div class="reservation-condition-edit__section">
                                    <div class="reservation-condition-edit__section-title">
                                        <Icon icon="material-symbols:settings-video-camera-outline-rounded" width="20px" height="20px" class="mr-1" />
                                        録画設定
                                    </div>
                                    <div class="reservation-condition-edit__section-description">
                                        この条件で自動追加される録画予約に適用される録画設定です。
                                    </div>
                                    <ReservationRecordingSettings
                                        :key="recordSettingsKey"
                                        :reservation="settingsReservationStub"
                                        :presets="presets"
                                        :has-changes="hasRecordSettingsChanges"
                                        @update-settings="recordSettings = $event"
                                        @changes-detected="hasRecordSettingsChanges = $event" />
                                </div>

                                <div class="reservation-condition-edit__footer">
                                    <v-btn variant="text" color="text" @click="$router.push('/reservations/conditions')">キャンセル</v-btn>
                                    <v-btn color="secondary" variant="flat" :loading="isSaving" @click="save">
                                        <Icon icon="fluent:save-20-regular" width="20px" height="20px" class="mr-1" />
                                        {{ isNew ? '登録する' : '保存する' }}
                                    </v-btn>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </main>
    </div>
</template>
<script lang="ts" setup>

import { computed, onMounted, ref, toRaw } from 'vue';
import { useRoute, useRouter } from 'vue-router';

import Breadcrumbs from '@/components/Breadcrumbs.vue';
import HeaderBar from '@/components/HeaderBar.vue';
import Navigation from '@/components/Navigation.vue';
import ProgramSearchFilters from '@/components/Programs/ProgramSearchFilters.vue';
import ReservationRecordingSettings from '@/components/Reservations/ReservationRecordingSettings.vue';
import SPHeaderBar from '@/components/SPHeaderBar.vue';
import Message from '@/message';
import { ILiveChannel } from '@/services/Channels';
import Programs, { IProgram, IProgramSearchCondition } from '@/services/Programs';
import ReservationConditions from '@/services/ReservationConditions';
import Reservations, { IRecordSettings, IRecordSettingsPresets, IReservation } from '@/services/Reservations';
import useChannelsStore from '@/stores/ChannelsStore';
import { ProgramUtils } from '@/utils';

const route = useRoute();
const router = useRouter();
const channelsStore = useChannelsStore();

// 編集対象の条件 ID (新規作成時は null)
const conditionId = computed<number | null>(() => {
    const id = Number.parseInt(route.params.id as string, 10);
    return Number.isFinite(id) && id > 0 ? id : null;
});
const isNew = computed(() => conditionId.value === null);

const createDefaultSearchCondition = (): IProgramSearchCondition => {
    return {
        is_enabled: true,
        keyword: '',
        exclude_keyword: '',
        note: '',
        is_title_only: false,
        is_case_sensitive: false,
        is_fuzzy_search_enabled: false,
        is_regex_search_enabled: false,
        service_ranges: null,
        genre_ranges: null,
        is_exclude_genre_ranges: false,
        date_ranges: null,
        is_exclude_date_ranges: false,
        duration_range_min: null,
        duration_range_max: null,
        broadcast_type: 'All',
        // 自動予約では、同じチャンネルで同名番組を録画済みなら新しい予約を無効状態で登録する (再放送の二重録画を避ける)
        duplicate_title_check_scope: 'SameChannelOnly',
        duplicate_title_check_period_days: 6,
    };
};

// 番組検索条件
const searchCondition = ref<IProgramSearchCondition>(createDefaultSearchCondition());

// 予約ドロワーの「シリーズ予約」から引き継いだ元番組名 (表示用)
const prefillSourceTitle = ref<string | null>(null);
const prefillChannelName = ref<string | null>(null);

// クエリパラメータ (keyword / network_id / transport_stream_id / service_id) から条件を初期化する
const applyQueryPrefill = () => {
    const keyword = typeof route.query.keyword === 'string' ? route.query.keyword.trim() : '';
    if (keyword === '') return;
    searchCondition.value.keyword = keyword;
    // 作品名で各話を拾う用途なので、説明文に作品名が出るだけの番宣・特集を拾わないよう番組名のみを対象にする
    searchCondition.value.is_title_only = true;
    const network_id = Number.parseInt(route.query.network_id as string, 10);
    const transport_stream_id = Number.parseInt(route.query.transport_stream_id as string, 10);
    const service_id = Number.parseInt(route.query.service_id as string, 10);
    if (Number.isFinite(network_id) && Number.isFinite(transport_stream_id) && Number.isFinite(service_id)) {
        searchCondition.value.service_ranges = [{ network_id, transport_stream_id, service_id }];
    }
    prefillSourceTitle.value = typeof route.query.source_title === 'string' ? route.query.source_title : keyword;
    prefillChannelName.value = typeof route.query.channel_name === 'string' ? route.query.channel_name : null;
};

// 録画設定
const recordSettings = ref<IRecordSettings | null>(null);
const presets = ref<IRecordSettingsPresets | null>(null);
const hasRecordSettingsChanges = ref(false);
// ReservationRecordingSettings は初期値を setup 時にコピーするため、読み込み完了時にキーを変えて再生成する
const recordSettingsKey = ref(0);

// ReservationRecordingSettings は録画予約を前提にしているため、録画設定だけを持つダミーの予約を渡す
const settingsReservationStub = computed<IReservation>(() => {
    return {
        id: -1,
        channel: null as unknown as IReservation['channel'],
        program: null as unknown as IReservation['program'],
        is_recording_in_progress: false,
        recording_availability: 'Full',
        comment: '',
        scheduled_recording_file_name: '',
        estimated_recording_file_size: 0,
        record_settings: recordSettings.value ?? ({} as IRecordSettings),
    };
});

const isLoading = ref(true);
const isSaving = ref(false);

// プレビュー (条件に一致する番組)
const isSearching = ref(false);
const hasPreviewed = ref(false);
const previewPrograms_ = ref<IProgram[]>([]);
const previewTotal = ref(0);
const PREVIEW_LIMIT = 20;

const allChannels = computed<ILiveChannel[]>(() => {
    return Object.values(channelsStore.channels_list).flat();
});

const channelMap = computed(() => {
    return new Map(allChannels.value.map((channel) => [channel.id, channel]));
});

const getChannelName = (program: IProgram): string => {
    return channelMap.value.get(program.channel_id)?.name ?? program.channel_id;
};

// 現在の番組表で条件に一致する番組を検索して表示する
const previewPrograms = async () => {
    isSearching.value = true;
    try {
        const result = await Programs.searchPrograms(searchCondition.value);
        hasPreviewed.value = true;
        if (result !== null) {
            const sorted = [...result.programs].sort((a, b) => new Date(a.start_time).getTime() - new Date(b.start_time).getTime());
            previewPrograms_.value = sorted.slice(0, PREVIEW_LIMIT);
            previewTotal.value = result.total;
        } else {
            previewPrograms_.value = [];
            previewTotal.value = 0;
        }
    } finally {
        isSearching.value = false;
    }
};

// 入力内容を検証する (EDCB 側で全番組にマッチする条件を登録してしまうのを防ぐ)
const validate = (): boolean => {
    const condition = searchCondition.value;
    const hasKeyword = condition.keyword.trim() !== '';
    const hasGenre = condition.genre_ranges !== null && condition.genre_ranges.length > 0;
    const hasChannel = condition.service_ranges !== null && condition.service_ranges.length > 0;
    if (!hasKeyword && !hasGenre && !hasChannel) {
        Message.error('検索キーワード・ジャンル・チャンネルのいずれかを指定してください。');
        return false;
    }
    if (!hasKeyword && !(hasGenre && hasChannel)) {
        Message.error('キーワードなしの条件は、ジャンルとチャンネルの両方を指定してください。(該当番組が多くなりすぎるため)');
        return false;
    }
    if (recordSettings.value === null) {
        Message.error('録画設定を読み込めていません。ページを再読み込みしてください。');
        return false;
    }
    return true;
};

const save = async () => {
    if (isSaving.value || !validate()) return;
    isSaving.value = true;
    try {
        const program_search_condition = structuredClone(toRaw(searchCondition.value));
        program_search_condition.keyword = program_search_condition.keyword.trim();
        program_search_condition.exclude_keyword = program_search_condition.exclude_keyword.trim();
        const record_settings = structuredClone(toRaw(recordSettings.value!));

        if (conditionId.value === null) {
            const success = await ReservationConditions.addReservationCondition(program_search_condition, record_settings);
            if (!success) return;
            Message.success('キーワード自動予約の条件を登録しました。条件に一致する番組は順次予約されます。');
        } else {
            const updated = await ReservationConditions.updateReservationCondition(conditionId.value, program_search_condition, record_settings);
            if (updated === null) return;
            Message.success('キーワード自動予約の条件を保存しました。');
        }
        hasRecordSettingsChanges.value = false;
        await router.push('/reservations/conditions');
    } finally {
        isSaving.value = false;
    }
};

onMounted(async () => {
    const [, loadedPresets, defaultRecordSettings] = await Promise.all([
        channelsStore.update(),
        Reservations.fetchRecordingPresets(),
        Reservations.fetchDefaultRecordSettings(),
    ]);
    presets.value = loadedPresets;

    if (conditionId.value !== null) {
        const condition = await ReservationConditions.fetchReservationCondition(conditionId.value);
        if (condition === null) {
            await router.replace('/reservations/conditions');
            return;
        }
        searchCondition.value = structuredClone(toRaw(condition.program_search_condition));
        recordSettings.value = structuredClone(toRaw(condition.record_settings));
    } else {
        recordSettings.value = structuredClone(toRaw(defaultRecordSettings));
        applyQueryPrefill();
    }
    recordSettingsKey.value += 1;
    isLoading.value = false;

    // 番組から引き継いだ条件は、そのまま一致する番組を確認できるよう自動で検索する
    if (prefillSourceTitle.value !== null) {
        await previewPrograms();
    }
});

</script>
<style lang="scss" scoped>

.reservation-condition-edit-container-wrapper {
    display: flex;
    flex-direction: column;
    width: 100%;
    min-width: 0;
}

.reservation-condition-edit-container {
    display: flex;
    flex-direction: column;
    width: 100%;
    height: 100%;
    padding: 20px;
    margin: 0 auto;
    min-width: 0;
    max-width: 1280px;

    @include smartphone-horizontal {
        padding: 16px 20px !important;
    }
    @include smartphone-horizontal-short {
        padding: 16px !important;
    }
    @include smartphone-vertical {
        padding: 8px 8px 20px !important;
    }
}

.reservation-condition-edit {
    display: flex;
    flex-direction: column;
    width: 100%;

    &__header {
        display: flex;
        align-items: center;
        justify-content: space-between;
        gap: 12px;
        flex-wrap: wrap;
    }

    &__title {
        display: flex;
        align-items: center;
        font-size: 22px;
        font-weight: 700;
        margin: 0;
        @include smartphone-vertical {
            font-size: 19px;
        }
    }

    &__title-back {
        display: flex;
        align-items: center;
        justify-content: center;
        width: 36px;
        height: 36px;
        margin-left: -8px;
        margin-right: 2px;
        border-radius: 50%;
        cursor: pointer;
    }

    &__actions {
        display: flex;
        align-items: center;
        gap: 8px;
    }

    &__loading {
        display: flex;
        justify-content: center;
        padding: 60px 0;
        color: rgb(var(--v-theme-text-darken-1));
    }

    &__prefill {
        display: flex;
        align-items: flex-start;
        gap: 10px;
        margin-top: 16px;
        padding: 12px 14px;
        border-radius: 10px;
        background: rgba(var(--v-theme-secondary), 0.12);
        font-size: 13px;
        line-height: 1.7;
    }
    &__prefill-icon {
        flex-shrink: 0;
        margin-top: 2px;
        color: rgb(var(--v-theme-secondary));
    }

    &__body {
        display: grid;
        grid-template-columns: minmax(0, 1fr) minmax(0, 1fr);
        gap: 24px;
        margin-top: 20px;

        @media (max-width: 1100px) {
            grid-template-columns: minmax(0, 1fr);
        }
    }

    &__search {
        min-width: 0;
        // ProgramSearchFilters はサイドバー用の幅指定を持つため、ここでは親幅いっぱいに広げる
        :deep(.program-search-filters) {
            width: 100%;
            max-width: none;
        }
    }

    &__side {
        display: flex;
        flex-direction: column;
        gap: 20px;
        min-width: 0;
    }

    &__section {
        padding: 16px;
        border-radius: 12px;
        background: rgb(var(--v-theme-background-lighten-1));
    }

    &__section-header {
        display: flex;
        align-items: center;
        justify-content: space-between;
        gap: 12px;
        :deep(.v-switch) {
            flex: none;
        }
    }

    &__section-title {
        display: flex;
        align-items: center;
        font-size: 16px;
        font-weight: 700;
    }

    &__section-count {
        margin-left: 10px;
        font-size: 13px;
        font-weight: 400;
        color: rgb(var(--v-theme-text-darken-1));
    }

    &__section-description {
        margin-top: 6px;
        margin-bottom: 12px;
        font-size: 13px;
        line-height: 1.7;
        color: rgb(var(--v-theme-text-darken-1));
    }

    &__preview {
        display: flex;
        flex-direction: column;
        gap: 8px;
        margin-top: 10px;
        max-height: 420px;
        overflow-y: auto;
    }

    &__preview-item {
        padding: 8px 10px;
        border-radius: 8px;
        background: rgb(var(--v-theme-background));
    }

    &__preview-time {
        font-size: 12px;
        color: rgb(var(--v-theme-text-darken-1));
    }

    &__preview-channel {
        margin-left: 8px;
        padding: 1px 6px;
        border-radius: 4px;
        background: rgb(var(--v-theme-background-lighten-1));
        color: rgb(var(--v-theme-text));
    }

    &__preview-title {
        margin-top: 2px;
        font-size: 14px;
        font-weight: 700;
    }

    &__footer {
        display: flex;
        justify-content: flex-end;
        gap: 8px;
    }
}

</style>

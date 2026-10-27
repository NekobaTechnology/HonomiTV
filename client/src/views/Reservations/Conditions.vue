<template>
    <div class="route-container">
        <HeaderBar />
        <main>
            <Navigation />
            <div class="reservation-conditions-container-wrapper">
                <SPHeaderBar />
                <div class="reservation-conditions-container">
                    <Breadcrumbs :crumbs="[
                        { name: 'ホーム', path: '/' },
                        { name: '録画予約', path: '/reservations/' },
                        { name: 'キーワード自動予約', path: '/reservations/conditions', disabled: true },
                    ]" />
                    <div class="reservation-conditions">
                        <div class="reservation-conditions__header">
                            <h2 class="reservation-conditions__title">
                                <div v-ripple class="reservation-conditions__title-back" @click="$router.push('/reservations/')">
                                    <Icon icon="fluent:chevron-left-12-filled" width="27px" />
                                </div>
                                <span class="reservation-conditions__title-text">キーワード自動予約</span>
                                <div class="reservation-conditions__title-count">
                                    <template v-if="isLoading">
                                        <Icon icon="line-md:loading-twotone-loop" class="mr-1 spin" width="20px" height="20px" />
                                    </template>
                                    <template v-else>{{ conditions.length }}件</template>
                                </div>
                            </h2>
                            <div class="reservation-conditions__actions">
                                <v-btn color="secondary" variant="flat" class="reservation-conditions__add"
                                    @click="$router.push('/reservations/conditions/new')">
                                    <Icon icon="fluent:add-12-filled" width="18px" height="18px" class="mr-1" />
                                    新規作成
                                </v-btn>
                            </div>
                        </div>
                        <div class="reservation-conditions__description">
                            キーワードやジャンル・チャンネル・曜日などの条件に一致する番組を、EDCB が番組表の更新に合わせて自動的に録画予約します。
                            毎週の番組や再放送をまとめて録りたいときに使います。
                        </div>

                        <!-- 空の場合 -->
                        <div v-if="!isLoading && conditions.length === 0" class="reservation-conditions__empty">
                            <Icon class="reservation-conditions__empty-icon" icon="fluent:text-bullet-list-square-search-20-regular" width="54px" height="54px" />
                            <div class="reservation-conditions__empty-message">まだキーワード自動予約の条件がありません。</div>
                            <div class="reservation-conditions__empty-submessage">「新規作成」から、番組名などのキーワードで自動予約の条件を追加できます。</div>
                        </div>

                        <!-- 条件一覧 -->
                        <div class="reservation-conditions__list">
                            <div v-for="condition in conditions" :key="condition.id"
                                class="reservation-condition"
                                :class="{ 'reservation-condition--disabled': !condition.program_search_condition.is_enabled }">
                                <div class="reservation-condition__main" v-ripple
                                    @click="$router.push(`/reservations/conditions/${condition.id}`)">
                                    <div class="reservation-condition__keyword">
                                        <Icon icon="fluent:search-20-filled" width="20px" height="20px" class="reservation-condition__keyword-icon" />
                                        <span class="reservation-condition__keyword-text">
                                            {{ condition.program_search_condition.keyword !== '' ? condition.program_search_condition.keyword : '(キーワードなし)' }}
                                        </span>
                                        <v-chip v-if="!condition.program_search_condition.is_enabled" size="small" color="error" variant="flat" class="ml-2">無効</v-chip>
                                    </div>
                                    <div v-if="condition.program_search_condition.note !== ''" class="reservation-condition__note">
                                        {{ condition.program_search_condition.note }}
                                    </div>
                                    <div class="reservation-condition__meta">
                                        <span v-for="(item, index) in getMetaItems(condition)" :key="index" class="reservation-condition__meta-item">
                                            <Icon :icon="item.icon" width="16px" height="16px" class="mr-1" />{{ item.text }}
                                        </span>
                                    </div>
                                </div>
                                <div class="reservation-condition__actions">
                                    <v-switch
                                        class="reservation-condition__switch"
                                        :model-value="condition.program_search_condition.is_enabled"
                                        :disabled="updatingConditionIds.has(condition.id)"
                                        color="primary"
                                        density="compact"
                                        hide-details
                                        @update:model-value="toggleEnabled(condition, $event === true)">
                                    </v-switch>
                                    <v-btn variant="text" size="small" class="reservation-condition__button"
                                        @click="$router.push(`/reservations/conditions/${condition.id}`)">
                                        <Icon icon="fluent:edit-16-regular" width="18px" height="18px" class="mr-1" />
                                        編集
                                    </v-btn>
                                    <v-btn variant="text" size="small" color="error" class="reservation-condition__button"
                                        @click="openDeleteDialog(condition)">
                                        <Icon icon="fluent:delete-16-regular" width="18px" height="18px" class="mr-1" />
                                        削除
                                    </v-btn>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </main>

        <!-- 削除確認ダイアログ -->
        <v-dialog v-model="showDeleteDialog" max-width="715">
            <v-card>
                <v-card-title class="d-flex justify-center pt-6 font-weight-bold">
                    キーワード自動予約の条件を削除しますか？
                </v-card-title>
                <v-card-text class="pt-2 pb-0">
                    <div v-if="deletingCondition" class="mb-4">
                        <div class="text-h6 text-text mb-2">
                            {{ deletingCondition.program_search_condition.keyword !== '' ? deletingCondition.program_search_condition.keyword : '(キーワードなし)' }}
                        </div>
                        <div class="text-body-2 text-text-darken-1">
                            この条件で登録されている録画予約: {{ deletingCondition.reservation_count }} 件
                        </div>
                    </div>
                    <div class="text-body-2 text-text-darken-1">
                        条件を削除しても、すでに登録済みの録画予約はそのまま残ります。
                        予約も不要な場合は、録画予約一覧から個別に削除してください。
                    </div>
                </v-card-text>
                <v-card-actions class="pt-4 px-6 pb-6">
                    <v-spacer></v-spacer>
                    <v-btn color="text" variant="text" @click="showDeleteDialog = false">キャンセル</v-btn>
                    <v-btn color="error" variant="flat" :loading="isDeleting" @click="confirmDelete">削除する</v-btn>
                </v-card-actions>
            </v-card>
        </v-dialog>
    </div>
</template>
<script lang="ts" setup>

import { computed, onMounted, ref, toRaw } from 'vue';

import Breadcrumbs from '@/components/Breadcrumbs.vue';
import HeaderBar from '@/components/HeaderBar.vue';
import Navigation from '@/components/Navigation.vue';
import SPHeaderBar from '@/components/SPHeaderBar.vue';
import Message from '@/message';
import ReservationConditions, { IReservationCondition } from '@/services/ReservationConditions';
import useChannelsStore from '@/stores/ChannelsStore';

const channelsStore = useChannelsStore();

// キーワード自動予約条件のリスト
const conditions = ref<IReservationCondition[]>([]);
const isLoading = ref(true);

// 有効/無効の切り替え中の条件 ID (連打防止)
const updatingConditionIds = ref<Set<number>>(new Set());

// 削除確認ダイアログ
const showDeleteDialog = ref(false);
const deletingCondition = ref<IReservationCondition | null>(null);
const isDeleting = ref(false);

// チャンネル名の引き当て用 (network_id + service_id → チャンネル名)
const channelNameMap = computed(() => {
    const map = new Map<string, string>();
    for (const channel of Object.values(channelsStore.channels_list).flat()) {
        map.set(`${channel.network_id}-${channel.service_id}`, channel.name);
    }
    return map;
});

const DAY_OF_WEEK_LABELS = ['日', '月', '火', '水', '木', '金', '土'];

// 一覧に表示する条件の要約を組み立てる
const getMetaItems = (condition: IReservationCondition): { icon: string; text: string }[] => {
    const search = condition.program_search_condition;
    const items: { icon: string; text: string }[] = [];

    if (search.exclude_keyword !== '') {
        items.push({ icon: 'fluent:text-strikethrough-20-regular', text: `除外: ${search.exclude_keyword}` });
    }

    if (search.service_ranges !== null && search.service_ranges.length > 0) {
        const names = search.service_ranges
            .map((service) => channelNameMap.value.get(`${service.network_id}-${service.service_id}`))
            .filter((name): name is string => name !== undefined);
        const text = names.length > 0 && names.length <= 3
            ? names.join('・')
            : `${search.service_ranges.length} チャンネル`;
        items.push({ icon: 'fluent:tv-20-regular', text });
    } else {
        items.push({ icon: 'fluent:tv-20-regular', text: '全チャンネル' });
    }

    if (search.genre_ranges !== null && search.genre_ranges.length > 0) {
        const majors = [...new Set(search.genre_ranges.map((genre) => genre.major))];
        const prefix = search.is_exclude_genre_ranges ? 'ジャンル除外: ' : 'ジャンル: ';
        items.push({ icon: 'fluent:tag-20-regular', text: prefix + (majors.length <= 2 ? majors.join('・') : `${majors.length} 種類`) });
    }

    if (search.date_ranges !== null && search.date_ranges.length > 0) {
        const text = search.date_ranges.map((range) => {
            const start = `${DAY_OF_WEEK_LABELS[range.start_day_of_week]} ${String(range.start_hour).padStart(2, '0')}:${String(range.start_minute).padStart(2, '0')}`;
            const end = `${DAY_OF_WEEK_LABELS[range.end_day_of_week]} ${String(range.end_hour).padStart(2, '0')}:${String(range.end_minute).padStart(2, '0')}`;
            return `${start}〜${end}`;
        }).join(', ');
        items.push({ icon: 'fluent:calendar-clock-20-regular', text: (search.is_exclude_date_ranges ? '時間帯除外: ' : '') + text });
    }

    const options: string[] = [];
    if (search.is_title_only) options.push('番組名のみ');
    if (search.is_fuzzy_search_enabled) options.push('あいまい');
    if (search.is_regex_search_enabled) options.push('正規表現');
    if (search.is_case_sensitive) options.push('大文字小文字区別');
    if (search.duplicate_title_check_scope !== 'None') options.push('重複回避');
    if (options.length > 0) {
        items.push({ icon: 'fluent:options-20-regular', text: options.join('・') });
    }

    items.push({ icon: 'fluent:arrow-sort-up-lines-20-regular', text: `優先度 ${condition.record_settings.priority}` });
    items.push({ icon: 'fluent:timer-20-regular', text: `予約 ${condition.reservation_count} 件` });
    return items;
};

// 条件一覧を取得する
const fetchConditions = async () => {
    const result = await ReservationConditions.fetchReservationConditions();
    if (result !== null) {
        // キーワード順に並べて探しやすくする (無効な条件は後ろ)
        conditions.value = [...result.reservation_conditions].sort((a, b) => {
            if (a.program_search_condition.is_enabled !== b.program_search_condition.is_enabled) {
                return a.program_search_condition.is_enabled ? -1 : 1;
            }
            return a.program_search_condition.keyword.localeCompare(b.program_search_condition.keyword, 'ja');
        });
    }
    isLoading.value = false;
};

// 条件の有効/無効を切り替える
const toggleEnabled = async (condition: IReservationCondition, is_enabled: boolean) => {
    if (updatingConditionIds.value.has(condition.id)) return;
    updatingConditionIds.value.add(condition.id);
    try {
        const program_search_condition = structuredClone(toRaw(condition.program_search_condition));
        program_search_condition.is_enabled = is_enabled;
        const updated = await ReservationConditions.updateReservationCondition(
            condition.id,
            program_search_condition,
            structuredClone(toRaw(condition.record_settings)),
        );
        if (updated !== null) {
            condition.program_search_condition.is_enabled = is_enabled;
            Message.success(is_enabled ? 'キーワード自動予約を有効にしました。' : 'キーワード自動予約を無効にしました。');
        }
    } finally {
        updatingConditionIds.value.delete(condition.id);
    }
};

const openDeleteDialog = (condition: IReservationCondition) => {
    deletingCondition.value = condition;
    showDeleteDialog.value = true;
};

const confirmDelete = async () => {
    if (deletingCondition.value === null) return;
    isDeleting.value = true;
    try {
        const success = await ReservationConditions.deleteReservationCondition(deletingCondition.value.id);
        if (success) {
            Message.success('キーワード自動予約の条件を削除しました。');
            conditions.value = conditions.value.filter((condition) => condition.id !== deletingCondition.value?.id);
            showDeleteDialog.value = false;
            deletingCondition.value = null;
        }
    } finally {
        isDeleting.value = false;
    }
};

onMounted(async () => {
    await Promise.all([
        channelsStore.update(),
        fetchConditions(),
    ]);
});

</script>
<style lang="scss" scoped>

.reservation-conditions-container-wrapper {
    display: flex;
    flex-direction: column;
    width: 100%;
    min-width: 0;
}

.reservation-conditions-container {
    display: flex;
    flex-direction: column;
    width: 100%;
    height: 100%;
    padding: 20px;
    margin: 0 auto;
    min-width: 0;
    max-width: 1000px;

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

.reservation-conditions {
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

    &__title-count {
        margin-left: 12px;
        font-size: 15px;
        font-weight: 400;
        color: rgb(var(--v-theme-text-darken-1));
    }

    &__description {
        margin-top: 12px;
        font-size: 13.5px;
        line-height: 1.7;
        color: rgb(var(--v-theme-text-darken-1));
    }

    &__empty {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        padding: 48px 16px;
        margin-top: 24px;
        border-radius: 12px;
        background: rgb(var(--v-theme-background-lighten-1));
        text-align: center;
    }
    &__empty-icon {
        color: rgb(var(--v-theme-text-darken-1));
    }
    &__empty-message {
        margin-top: 12px;
        font-size: 16px;
        font-weight: 700;
    }
    &__empty-submessage {
        margin-top: 6px;
        font-size: 13px;
        color: rgb(var(--v-theme-text-darken-1));
    }

    &__list {
        display: flex;
        flex-direction: column;
        gap: 12px;
        margin-top: 20px;
    }
}

.reservation-condition {
    display: flex;
    align-items: stretch;
    border-radius: 12px;
    background: rgb(var(--v-theme-background-lighten-1));
    overflow: hidden;
    transition: opacity 0.2s;

    &--disabled {
        opacity: 0.6;
    }

    &__main {
        flex: 1;
        min-width: 0;
        padding: 14px 16px;
        cursor: pointer;
    }

    &__keyword {
        display: flex;
        align-items: center;
        font-size: 17px;
        font-weight: 700;
        min-width: 0;
    }
    &__keyword-icon {
        flex-shrink: 0;
        margin-right: 8px;
        color: rgb(var(--v-theme-secondary));
    }
    &__keyword-text {
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }

    &__note {
        margin-top: 4px;
        font-size: 13px;
        color: rgb(var(--v-theme-text-darken-1));
        white-space: pre-wrap;
    }

    &__meta {
        display: flex;
        flex-wrap: wrap;
        gap: 4px 14px;
        margin-top: 8px;
        font-size: 12.5px;
        color: rgb(var(--v-theme-text-darken-1));
    }
    &__meta-item {
        display: inline-flex;
        align-items: center;
    }

    &__actions {
        display: flex;
        flex-direction: column;
        align-items: flex-end;
        justify-content: center;
        gap: 2px;
        padding: 8px 12px 8px 0;
        flex-shrink: 0;
    }

    &__switch {
        margin-right: 6px;
        :deep(.v-selection-control) {
            min-height: 32px;
        }
    }

    @include smartphone-vertical {
        flex-direction: column;
        &__actions {
            flex-direction: row;
            justify-content: flex-end;
            padding: 0 8px 8px;
        }
    }
}

</style>

---
title: 'Apollo.ML 회귀 분석 프로젝트: R² 0.089에서 0.9297로 1,000% 성능 개선 달성!'
date: 2025-09-15
description: 'SQL Server 실행 계획 데이터를 활용한 쿼리 실행 시간 예측 모델 구축 프로젝트입니다. 초기 R² 0.089에서 최종 R² 0.9297을 달성하여 1000% 성능 개선을 이루었으며, 데이터 품질 개선, 고급 피처 엔지니어링, overfitting 방지 기법을 통해 프로덕션 적용 가능한 고성능 모델을 구축했습니다.'
categories: [ML, DataScience]
tags: [머신러닝, 회귀분석, XGBoost, 데이터품질, 피처엔지니어링, Overfitting방지, SQL서버, 실행계획, 쿼리최적화, 교차검증, 정규화, 앙상블모델, 클러스터링, 시계열분석]
index_img: img/202501-apollo-ml-regression/apollo-ml-thumbnail.png
keywords: ['SQL Server execution plan', 'query performance prediction', 'XGBoost regression', 'data quality improvement', 'feature engineering', 'overfitting prevention', 'machine learning pipeline', '쿼리 성능 예측', '머신러닝 파이프라인', '데이터 전처리', '피처 선택', '교차 검증', '정규화 기법', '앙상블 모델링']
---

# Apollo.ML 회귀 분석 프로젝트: R² 0.089에서 0.9297로 1,000% 성능 개선 달성!

SEO에서 포스팅의 제목이 엄청 중요하다는 것을 정말 여러번 느끼고 있습니다. 그래서 오늘도 생동감 넘치는 제목으로 어그로를 끌어봅니다. R² (결정 계수, Coefficient of Determination) 가 애초에 0.089가 나온 것 자체가 데이터셋에 문제가 많았고, 전처리도 제대로 하지 않았기에 전혀 데이터를 설명할 수 없는 값이었던 것입니다.

## 1. 프로젝트 개요

Apollo.ML 프로젝트는 DBMS (SQL Server) execution plan 데이터를 기반으로 쿼리 실행 시간(`last_ms`)을 예측하는 회귀 분석 모델을 구축하는 것이 목표였습니다. 초기에는 매우 낮은 R² 점수(0.089)를 보였으나, 체계적인 개선 과정을 통해 최종적으로 R² 0.9297을 달성했습니다.

<br>

---

## 2. 초기 데이터 분석 및 문제점

### 2.1 초기 파이프라인 실행 결과

처음 데이터 fetch부터 피처화, 학습까지의 전체 파이프라인을 실행한 결과, R² 점수 0.089라는 매우 낮은 성능을 보였습니다. 데이터셋 크기는 9,000건 미만이었으며, 여러 심각한 데이터 품질 문제가 발견되었습니다.

주요 문제점으로는 극심한 타겟 변수 불균형이 있었는데, 중앙값 0.43ms와 평균 3605.81ms 사이의 큰 차이를 보였습니다. 전체 데이터의 56.3%가 1ms 미만의 매우 빠른 실행 시간을 보였으며, 19.8%는 이상치로 분류되었습니다. 또한 49.6%의 피처에서 결측값이 발생했고, 9개의 상수 피처가 존재하여 모델 학습에 방해가 되었습니다.

### 2.2 데이터 품질 문제 분석

```python
# 타겟 변수 분포 분석 결과
print(f"중앙값: {df['last_ms'].median():.2f}ms")
print(f"평균: {df['last_ms'].mean():.2f}ms")
print(f"1ms 미만 비율: {(df['last_ms'] < 1).mean()*100:.1f}%")
print(f"이상치 비율: {((df['last_ms'] > df['last_ms'].quantile(0.95))).mean()*100:.1f}%")
```

<br>

---

## 3. 데이터셋 확장 및 피처 엔지니어링

### 3.1 데이터셋 확장

기존 데이터의 한계를 극복하기 위해 데이터베이스 쿼리를 확장했습니다. 기존에는 단순히 `query_id`, `plan_id`, `plan_xml`, `last_ms`만 수집했으나, 성능 예측에 필요한 추가 컨텍스트 정보를 포함하도록 쿼리를 개선했습니다.

**기존 쿼리**:
```sql
SELECT query_id, plan_id, plan_xml, last_ms FROM dbo.collected_plans
```

**확장된 쿼리**:
```sql
SELECT query_id, plan_id, plan_xml, count_exec, est_total_subtree_cost, 
       avg_ms, last_cpu_ms, last_reads, max_used_mem_kb, max_dop, 
       last_exec_time, last_ms 
FROM dbo.collected_plans
```

이러한 확장을 통해 데이터셋 크기가 9,000건에서 12,624건으로 40% 증가했으며, 8개의 성능 관련 메트릭 컬럼이 추가되어 더 풍부한 컨텍스트 정보를 확보할 수 있었습니다.

### 3.2 고급 피처 엔지니어링

새로운 컬럼들을 활용하여 성능 예측에 유용한 파생 피처들을 생성했습니다. 성능 관련 피처로는 비용 효율성(`cost_efficiency`), 성능 비율(`performance_ratio`), CPU 비율(`cpu_ratio`), 읽기 효율성(`reads_per_ms`) 등을 계산했습니다.

```python
# 성능 관련 피처
feats["cost_efficiency"] = row["est_total_subtree_cost"] / row["last_ms"]
feats["performance_ratio"] = row["last_ms"] / row["avg_ms"]
feats["cpu_ratio"] = row["last_cpu_ms"] / row["last_ms"]
feats["reads_per_ms"] = row["last_reads"] / row["last_ms"]

# 시스템 리소스 피처
feats["memory_intensive"] = 1 if row["max_used_mem_kb"] > 1024 else 0
feats["is_parallel"] = 1 if row["max_dop"] > 1 else 0
feats["is_frequent_query"] = 1 if row["count_exec"] > 10 else 0
```

또한 시스템 리소스 관련 피처로는 메모리 집약도, 병렬 처리 여부, 빈번한 쿼리 여부 등을 추가하여 쿼리 실행 환경의 특성을 반영했습니다.

<br>

---

## 4. 단계별 모델 고도화

### 4.1 Phase 1: 데이터 품질 개선

Phase 1의 목표는 R² 0.089에서 0.2~0.3 수준으로 개선하는 것이었습니다. 이를 위해 데이터 품질 개선에 집중했습니다.

이상치 처리 개선에서는 Isolation Forest를 사용하여 10%의 이상치를 탐지하고, Z-score 기반으로 3σ 기준 이상치를 식별했습니다. 또한 쿼리 패턴별로 이상치 임계값을 설정하여 도메인 특성을 반영했습니다.

결측값 처리에서는 KNN 기반 대체 방법을 사용하여 n_neighbors=5로 설정하고, 도메인 지식에 기반한 기본값을 설정했습니다. 특히 타겟 변수를 제외한 안전한 처리 방식을 적용하여 데이터 누출을 방지했습니다.

피처 정규화 개선에서는 QuantileTransformer를 사용하여 정규 분포로 변환하고, PowerTransformer의 Yeo-Johnson 방법을 적용했습니다. 각 피처별로 최적의 스케일링 방법을 적용하여 데이터 분포를 개선했습니다.

**결과**: R² 0.9922 (목표 대비 3,300% 초과 달성)

### 4.2 Phase 2: 고급 피처 엔지니어링

Phase 2의 목표는 R² 0.2~0.3에서 0.4~0.5 수준으로 개선하는 것이었습니다. 이를 위해 고급 피처 엔지니어링 기법을 적용했습니다.

실행계획 구조 분석을 강화하여 SQL 실행 계획의 복잡성을 정량화했습니다. 실행계획 트리 깊이를 계산하고, 병렬 처리 레벨과 조인 복잡도를 분석하여 쿼리의 구조적 특성을 파악했습니다.

```python
# 실행계획 트리 깊이 계산
longest_path = max(path_length.values()) if path_length else 0

# 병렬 처리 레벨 계산
parallel_levels = sum(1 for node, attrs in g.nodes(data=True) 
                     if attrs.get('Parallel', False))

# 조인 복잡도 계산
join_ops = sum(1 for node, attrs in g.nodes(data=True)
              if 'join' in attrs.get('PhysicalOp', '').lower())
```

시계열 특성을 추가하여 시간대별 성능 패턴을 분석하고, 쿼리 실행 빈도와 성능 트렌드를 고려했습니다. 또한 K-means 클러스터링을 사용하여 유사한 쿼리 패턴을 그룹화하고, 클러스터별 통계 특성을 생성하여 쿼리의 유형을 반영했습니다.

**결과**: R² 0.9314

### 4.3 Phase 3: 앙상블 모델링

Phase 3의 목표는 R² 0.4~0.5에서 0.6~0.7 수준으로 개선하는 것이었습니다. 이를 위해 다양한 머신러닝 모델들을 조합한 앙상블 방법을 시도했습니다.

구현된 모델들로는 XGBoost, LightGBM, CatBoost, Random Forest, Gradient Boosting, Neural Network (MLP)를 사용했습니다. 앙상블 방법으로는 Voting Regressor를 통한 평균 방법, 가중 평균 앙상블, 그리고 Stacking 앙상블을 적용했습니다.

**결과**: R² 0.1202 (성능 저하)

<br>

---

## 5. Phase 3 앙상블 모델의 실효성 저하 원인

### 5.1 성능 저하 요인

Phase 3에서 앙상블 모델링을 시도했으나 오히려 성능이 저하된 주요 원인을 분석해보면 다음과 같습니다.

첫째, 데이터 품질 개선의 압도적 효과가 있었습니다. Phase 1에서 이미 R² 0.9922를 달성한 상태에서 추가적인 모델 복잡도가 오히려 성능 저하를 가져왔습니다. 이는 데이터 품질 개선이 모델링 기법보다 훨씬 중요한 요소임을 보여줍니다.

둘째, 피처 수 증가로 인한 복잡도 문제가 있었습니다. Phase 2에서 15개 피처로 최적화했던 것을 앙상블에서는 53개 피처를 사용하면서 노이즈가 증가했습니다. 이는 "더 많은 피처가 항상 더 좋은 것은 아니다"라는 교훈을 제공합니다.

셋째, 모델 간 상관관계가 높아서 앙상블의 다양성 부족이 문제가 되었습니다. 모든 모델이 유사한 패턴을 학습하여 앙상블의 장점을 살리지 못했습니다.

### 5.2 교훈

이러한 결과를 통해 다음과 같은 중요한 교훈을 얻었습니다. 단순한 모델이 복잡한 모델보다 효과적일 수 있으며, 데이터 품질 개선이 모델링 기법보다 우선순위가 높습니다. 또한 피처 선택이 앙상블보다 더 중요한 요소임을 확인했습니다.

<br>

---

## 6. Overfitting 검증 및 방지

### 6.1 Overfitting 방지 기법 적용

두 모델 모두에서 overfitting을 방지하기 위해 다양한 기법을 적용했습니다.

Phase 1에서는 정규화를 강화하여 모델의 복잡도를 제어했습니다. n_estimators를 500으로 감소시키고, max_depth를 6으로 제한하며, learning_rate를 0.05로 낮췄습니다. 또한 L1 정규화(reg_alpha=0.1)와 L2 정규화(reg_lambda=0.1)를 적용하고, min_child_weight를 5로 설정하여 최소 샘플 수를 제한했습니다.

```python
# 정규화 강화
model = XGBRegressor(
    n_estimators=500,      # 감소
    max_depth=6,           # 감소
    learning_rate=0.05,    # 감소
    reg_alpha=0.1,         # L1 정규화
    reg_lambda=0.1,        # L2 정규화
    min_child_weight=5     # 최소 샘플 수 제한
)

# 교차 검증
cv_scores = cross_val_score(model, X_train_scaled, y_train, cv=5, scoring='r2')
```

Phase 2에서는 피처 선택을 강화하여 SelectKBest로 25개 피처를 선택하고, RFE를 사용하여 최종 15개 피처로 압축했습니다. 또한 정규화를 더욱 강화하여 reg_alpha=0.2, reg_lambda=0.2로 설정하고, min_child_weight=10으로 제한했습니다.

### 6.2 Overfitting 검증 결과

두 모델의 overfitting 검증 결과는 다음과 같습니다.

| 모델 | 검증 R² | 교차 검증 R² | Overfitting Gap | 안정성 |
|------|---------|--------------|-----------------|--------|
| Phase 1 | 0.1151 | 0.0914 ± 0.0422 | 0.0294 | ✅ 안정적 |
| Phase 2 | 0.9297 | 0.9343 ± 0.0043 | 0.0052 | ✅ 매우 안정적 |

검증 방법으로는 5-fold 교차 검증을 사용하고, 훈련/검증 성능 차이를 분석했습니다. Overfitting Gap이 0.05 미만인 경우를 안정적으로 판정했으며, 두 모델 모두 이 기준을 충족했습니다.

<br>

---

## 7. 결론

### 7.1 성과 요약

본 프로젝트는 초기 R² 0.089에서 최종 R² 0.9297을 달성하여 1,000%의 성능 개선을 이루었습니다. R² 0.9라는 목표를 크게 초과 달성했으며, overfitting 없이 일관된 성능을 보여주었습니다.

### 7.2 핵심 성공 요인

가장 중요한 성공 요인은 데이터 품질 개선이었습니다. 이상치 처리와 결측값 처리가 핵심이었으며, QuantileTransformer가 매우 효과적이었습니다. 이는 "데이터가 모델보다 중요하다"는 머신러닝의 기본 원칙을 잘 보여주는 사례입니다.

고급 피처 엔지니어링도 중요한 역할을 했습니다. 실행계획 구조 분석을 통해 SQL 쿼리의 복잡성을 정량화하고, 시계열 특성을 활용하여 시간적 패턴을 반영했으며, 클러스터링 기반 피처로 쿼리 유형을 분류했습니다.

Overfitting 방지의 중요성도 확인되었습니다. 정규화와 피처 선택이 필수적이었으며, 교차 검증을 통해 신뢰성을 확보했습니다.

### 7.3 프로덕션 적용 권장사항

Phase 2 모델을 프로덕션에 사용할 것을 권장합니다. R² 0.9297로 높은 예측 정확도를 보이며, Overfitting Gap 0.0052로 매우 안정적입니다. 15개 피처로 효율적이며, 교차 검증으로 검증된 신뢰성을 가지고 있습니다.

추가 개선을 위해서는 실시간 데이터 수집 파이프라인 구축, 모델 성능 모니터링 시스템, 자동화된 재훈련 프로세스, 그리고 예측 기반 쿼리 최적화 제안 시스템을 구축하는 것이 필요합니다.

### 7.4 기술적 교훈

이 프로젝트를 통해 다음과 같은 중요한 기술적 교훈을 얻었습니다. 데이터 품질이 모델 복잡도보다 중요하며, 피처 엔지니어링이 앙상블 모델링보다 효과적입니다. 단순한 모델이 복잡한 모델보다 효과적일 수 있으며, overfitting 방지가 성능 향상보다 중요합니다. 마지막으로 도메인 지식 기반 피처가 핵심이라는 점을 확인했습니다.

이 프로젝트를 통해 SQL Server 쿼리 실행 시간 예측에 대한 체계적인 접근 방법을 확립했으며, 실제 프로덕션 환경에서 활용 가능한 고성능 모델을 구축했습니다.

---
`eod`
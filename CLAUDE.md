# SiO2 유리 기판 위 Cu 증착 MD 미니프로젝트

> 이 파일은 새 세션에서 맥락을 즉시 복원하기 위한 인계 문서다.
> 새 대화를 시작하면 이 파일부터 읽을 것.

## 0. 목적과 핵심 논점

취업 포트폴리오/면접 자료 (코닝 8/3 전화인터뷰, ASM, LG디스플레이 — 셋 다 원자 수준
재료 시뮬레이션 인력). LAMMPS 사용. 3일 내 완료 목표.

**핵심 논점**: 비정질 SiO2 표면에 Cu를 증착하면 섬 형태로 뭉치는(Volmer-Weber) 거동이
예상되는데, 그게

- **(a)** MD 시간 스케일 한계로 표면확산이 부족해서인지
- **(b)** Cu-SiO2 계면 결합이 실제로 약해서인지

를 구분해 보이는 것이 목표. 단순 증착 실행이 아니라 **이 둘을 가르는 판정 설계**가
결과물의 핵심이다.

## 1. 확정 사항 (재논의 불필요)

### 1.1 기판 구조
- `init_struct/sio2_quenched.data` : BKS melt-quench 비정질 SiO2 벌크
- 2160 atoms = O(type1, q=-1.2) 1440 + Si(type2, q=+2.4) 720 (정확한 화학량론, 순전하 0)
- 박스 28.728 x 28.829 x 33.270 A, 밀도 2.607 g/cm3
- 평균 배위수 <CN(Si-O)> = 4.017, <CN(O-Si)> = 2.008 (r<2.0 A) — 결함 거의 없는 양질의 유리
- Si-O 결합 총 2892개

### 1.2 밀도 이슈 (규명 완료)
실험 a-SiO2 는 2.20 g/cm3 인데 이 구조는 2.607. 검증 결과:
- overlay 없는 별도 런도 2.566 -> 하이브리드 overlay 와 냉각속도 모두 원인 아님
- **진짜 원인은 buck/coul/long 의 10 A cutoff.** BKS 는 이 cutoff 에서 유리 밀도를
  과대평가하는 것이 문헌에 알려져 있음. 5.5 A 로 줄이면 밀도는 개선되나 탄성물성이 틀어짐
- 계면 역학이 중요한 본 과제에서는 **10 A 를 유지**하고, 모든 결론은 **조건 간 상대 비교로만** 해석

### 1.3 Tg 이슈 (규명 완료)
BKS 는 밀도 최대점이 ~4500 K(실험 ~1850 K)라 V-T 로 Tg 를 뽑으면 안 됨.
엔탈피 기반 Tg 2700~2800 K 는 BKS 문헌값(1 K/ps 에서 ~3000 K) 범위 내로 타당.

### 1.4 힘장
`common/ff_sio2.in` 에 단일 관리. melt-quench 입력과 파라미터가 **완전히 동일**해야 함.
- BKS: van Beest, Kramer, van Santen, PRL 64, 1955 (1990)
- Buckingham catastrophe 대응: ZBL(~r^-1)은 -C/r^6 을 못 이겨 무효였음.
  r^-12 LJ(WCA형)로 교체. O-O eps=3.0 sigma=1.50 / Si-O eps=4.0 sigma=1.10 (eV, A)
- `kspace_modify slab 3.0` 은 ff 파일이 아니라 **각 단계 스크립트에서** 선언
  (주기 상태에선 켜면 안 되고, 절단 전/후 두 상태를 모두 계산해야 하므로)

### 1.5 슬랩 형상
- z 박스 120 A (진공/물질 비는 제약이 아님 — LAMMPS 문서상 제약은 volfactor 뿐.
  실제 제약은 기하학: 0~34 슬랩 / ~34~65 Cu / 90~95 증착원 삽입영역 / 118 상단 반사벽)
- lateral 은 1x1 (28.7 A) 유지. 섬 크기가 박스에 제한된다는 점은 한계로 명시
- `change_box` 로 z 확장 후 boundary p p f. **remap 절대 금지**(좌표 스케일링됨)

## 2. 실행 환경

| 항목 | 값 |
|---|---|
| 노트북 (Claude 작업) | macOS, `~/projects/SiO2_Cu-deposition` |
| 데스크탑 (LAMMPS 실행) | Linux 12코어, `~/projects/SiO2_Cu-deposition`, 원격 접속 |
| 동기화 | git + GitHub `git@github.com:vkxlzptm/SiO2_Cu-deposition.git` (public) |
| LAMMPS | 23 Jun 2022 Update 1, conda env `lammps_env`, 바이너리 `lmp_serial` / `lmp_mpi` |
| MANYBODY (pair_style eam) | **설치됨** (검증 완료) |
| potentials | `/home/dhl/lammps/potentials`, `LAMMPS_POTENTIALS` 등록됨 |
| Cu EAM 선택 | **`Cu_mishin1.eam.alloy`** -> `pair_style eam/alloy`. Mishin et al., PRB 63, 224106 (2001). 표면·적층결함 에너지를 피팅에 포함해 gamma_Cu 재현이 중요한 본 과제에 적합 |

## 3. 디렉토리 구조와 단계 독립성 원칙

```
SiO2_Cu-deposition/
├── CLAUDE.md            이 파일
├── common/ff_sio2.in    BKS 힘장 (S1, S2 전용)
├── common/ff_full.in    위 + eam(Cu) + lj/cut(Cu-O, Cu-Si)   (S3 이후, 미작성)
├── init_struct/         sio2_quenched.data, in.sio2  (읽기전용 원본)
├── S1_slab/    in.slab     <- init_struct/          -> slab_raw.data
├── S2_relax/   in.relax    <- S1/slab_raw.data      -> slab_relaxed.data   ★분기점
├── S3_ff/      in.fftest   <- S2  (검증 전용)
├── S4_wadh/    in.wadh     <- S2  -> wadh_*.log
├── S5_dep/     in.deposit  <- S2  -> dep_T*_E*.data
├── S6_ctrl/    in.cuoncu   <- S2
└── analysis/
```

**원칙: 각 단계는 `read_data` 로 시작해 `write_data` 로 끝난다.** 단계 경계에서 상태가
완전히 파일로 직렬화되므로 어느 단계가 터져도 그 단계만 재실행하면 된다.
S2 가 분기점 — S4/S5/S6 은 전부 `slab_relaxed.data` 하나만 읽으므로 서로 독립이고 순서도 자유.
단계 **안쪽**은 `restart 10000 x.restart.a x.restart.b` 로 이어달리기 (thermostat 적분변수까지 보존).

S3 이후 Cu 타입 추가는 데이터 파일 재생성 없이:
`read_data ../S2_relax/slab_relaxed.data extra/atom/types 1`

## 4. 작업 방식 규칙 (사용자 지정)

- 파일 수정 전에 항상 텍스트로 제안하고 승인받을 것
- 계산량 큰 작업(실제 MD 실행 등)은 착수 전 확인받을 것
- 검증 안 된 수치를 지어내지 말 것. 문헌값은 출처와 함께, 불확실하면 불확실하다고 명시
- 사용자가 세운 가설도 데이터로 반증되면 그대로 말할 것
- 에이전트를 많이 쓰는 고비용 작업은 착수 전 확인받을 것
- 우선순위: S3(교차항+EAM)가 최대 리스크. 시간 부족 시 **S6 의 Cu-on-Cu 대조군만은 반드시 남길 것**
  (이게 빠지면 프로젝트의 핵심 논점 자체가 사라짐)

## 5. 진행 상태

| 단계 | 상태 | 비고 |
|---|---|---|
| S0 환경검증 | **완료** | MANYBODY 있음, Cu_mishin1.eam.alloy 확보 |
| S1 슬랩생성 | **재실행 대기** | 물리 판정 4/4 통과. `fix ave/chunk` 문법 오류로 중단 -> 수정 완료 |
| S2 표면이완 | 미착수 | |
| S3 힘장확장 | 조사 완료, 스크립트 미작성 | 아래 6절 참조 — **계획 수정 필요** |
| S4 부착일 | 미착수 | |
| S5 증착 | 미착수 | |
| S6 판정 | 미착수 | |
| S7 분석/문서화 | 미착수 | |

### S1 1차 실행 결과 (수정 전)
- 원자수 2160 (손실 0) / <CN(Si-O)> 3.9167 / <CN(O-Si)> 1.9583
- -> 끊긴 결합 **72개** (2892 - 2820). 사전 z평면 스캔 예측 "평균 70개"와 일치
- Pzz = -3892 bar (인장), Pxx = -1047, Pyy = -301 (셀 크기상 압력 요동 범위 내)
- PotEng = -41038.959 eV (pe/atom = -18.9995)
- PPPM: grid 24x24x150, relative force accuracy 1.01e-4 — 정상
- 슬랩의 **절대 압력값은 해석 금지** (부피에 진공 86 A 포함, 정규화가 자의적).
  S2 전후 변화량으로만 볼 것

## 6. S3 교차항 조사 결과 — 원래 계획 수정 필요

### 6.1 UFF 파라미터 (원문 Table I 직접 확인)
출처: Rappé, Casewit, Colwell, Goddard, Skiff, **JACS 114, 10024 (1992)**
(x = 원자 van der Waals **거리 r_min** [A], D = 우물깊이 [kcal/mol])

| UFF type | x | D |
|---|---|---|
| O_3   | 3.500 | 0.060 |
| O_3_z (제올라이트 O, theta0=146도) | 3.500 | 0.060 |
| Si3   | 4.295 | 0.402 |
| Cu3+1 | 3.495 | **0.005** |

혼합규칙: UFF 자체는 거리·우물깊이 **둘 다 기하평균** (eq 21b, 22). 논문은 산술평균(21a,
= Lorentz)도 언급하나 "somewhat problematic"이라 하고 자체 파라미터화엔 기하평균을 씀.
본 과제에서 둘의 차이는 Cu-Si 에서 0.5% 뿐이라 **수치적으로 무의미**.

### 6.2 LAMMPS `lj/cut` 입력값 (sigma = x / 2^(1/6), eps 를 eV 로)

| pair | sigma [A] | eps [eV] |
|---|---|---|
| Cu-O  | 3.11592 | 7.5109e-4 |
| Cu-Si | 3.45170 | 1.9441e-3 |

### 6.3 ★문제: baseline 이 터무니없이 약하다★
- eps(Cu-O) = 0.751 meV = **0.029 kT @ 300 K**
- 두 반무한 매질 -C/r^6 적분(Hamaker형) 추정: **W_adh ~ 0.012 J/m2**
- 원인: UFF 의 전이금속 vdW 파라미터는 피팅이 아니라 추정값이고, Cu3+1 의 D=0.005 는
  UFF 전체에서도 이례적으로 작음 (비교: Ni4+2 0.015, Co6+3 0.014, Zn3+2 0.124).
  화학적 Cu-O 결합(d-p 혼성)이 전혀 반영돼 있지 않음

**따라서 원래 계획한 "0.5x / 1x / 2x eps 스캔"은 세 조건 모두 비젖음 영역에 몰려
젖음/비젖음 경계를 찾지 못한다.** S4 가 (a)/(b) 판정의 기준선 역할을 못 하게 됨.

### 6.4 대안 앵커: DFT 문헌값으로 스캔을 보정
출처: **Nagao, Neaton, Ashcroft, arXiv:cond-mat/0304459 (2003)**, LDA-DFT,
Cu/alpha-cristobalite(001), 이상 분리일 W = (E_SiO2 + E_Cu - E_IF)/A

| 계면 종단 | W [J/m2] |
|---|---|
| OHOH (수산화, "wet" 증착 조건) | **0.331** |
| Si-terminated | **1.406** |
| O-terminated | **1.555** |
| OO-terminated (산소 과잉, 탈수산화 후) | **3.805** |

- OO-terminated 의 ~4 J/m2 는 Kriese, Moody, Gerberich, Acta Mater 46, 6623 (1998) 의
  100 nm Cu/SiO2 압입 실험값과 일치한다고 원논문이 명시
- 원논문 결론: Cu 의 나쁜 밀착은 **표면 수산화와 산소 결핍** 탓이며, 탈수산화하고
  산소 밀도를 높이면 "robust adhesion is entirely possible"

**이는 사용자가 세운 검증 앵커("baseline 에서 W_adh < 2*gamma_Cu 면 타당")를
단순화된 것으로 만든다.** W_adh 는 종단 화학에 따라 10배 이상 변하며, OO 종단은
2*gamma_Cu 를 넘길 수도 있다. 이 긴장은 사용자에게 보고했고 설계 반영 필요.

### 6.5 제안하는 스캔 재설계 (미승인)
임의 배율(0.5x/1x/2x) 대신 **DFT 사다리에 맞춰 eps 를 역보정**한다:

| 조건 | 목표 W_adh | 물리적 대응 | UFF 대비 배율(Hamaker 추정) |
|---|---|---|---|
| A | 0.33 | 수산화 종단 | ~27x |
| B | 1.4~1.6 | Si / O 종단 (표준) | ~114~126x |
| C | 3.8 | OO 종단 (탈수산화) | ~309x |

배율은 어디까지나 해석적 추정이므로, **실제로는 S4 에서 eps 를 로그 스캔하며
W_adh 를 직접 계산해 위 목표값에 맞추는 방식**으로 확정해야 한다.
이렇게 하면 각 조건이 실제 인용 가능한 표면 화학에 대응되어, 임의 배율 스캔보다
논증이 훨씬 강해진다. 2*gamma_Cu (Mishin EAM 으로 직접 계산) 가 어디에 떨어지는지가
젖음/비젖음 경계가 된다.

한계 및 future work: 이 근사는 Cu-O 화학결합을 등방 LJ 로 뭉뚱그린 것.
정공법은 DFT 라벨링 -> MLIP 학습으로 계면 상호작용을 정량화하는 것이며 문서에 남긴다.

## 7. git 워크플로

```bash
# 노트북 (스크립트 수정 후)
git add -A && git commit -m "..." && git push
# 데스크탑 (실행 전/후)
git pull && lmp_serial -in in.xxx -log log.xxx && git add -A && git commit -m "..." && git push
```
규칙: **작업 시작 전 `git pull`, 끝나면 `git push`.**
`.gitignore` 로 `*.lammpstrj`, `*.restart.*` 는 제외 (dump 는 데스크탑에만 둔다).
데스크탑에 SSH 키 등록이 아직 안 돼 있으면 push 불가 — `ssh-keygen -t ed25519` 후
공개키를 GitHub 에 등록하고 `git remote set-url origin git@github.com:...` 로 교체.

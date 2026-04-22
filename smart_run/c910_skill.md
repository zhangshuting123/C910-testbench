# C910 OpenC910 仿真系统更新说明

## 目录
1. [更新概述](#更新概述)
2. [smart_cfg.mk 更新详情](#smart_cfg_mk-更新详情)
3. [Makefile 更新详情](#makefile-更新详情)
4. [核心流程解析](#核心流程解析)
5. [新增功能使用指南](#新增功能使用指南)
6. [Makefile 技术要点](#makefile-技术要点)

---

## 更新概述

本次更新主要针对 **Verilog 测试用例（如 vfpu、iu）的批量编译和仿真支持**，实现了以下关键功能：

| 功能 | 原版 | 更新版 |
|------|------|--------|
| 默认仿真器 | iverilog | vcs |
| 波形dump | off | on |
| vfpu/iu 支持 | ❌ 无 | ✅ 支持 |
| 批量 .v 测试 | ❌ 手动指定 | ✅ 自动搜索 |
| 顺序编译模式 | ❌ 无 | ✅ SEQ_LIST |

---

## smart_cfg.mk 更新详情

### 1. 新增测试用例到 CASE_LIST

**位置**: Lines 16-33

```makefile
# 原版 (缺失 iu, vfpu)
CASE_LIST := \
      ISA_AMO \
      smoke_bus \
      ...
      sleep \

# 更新版 (新增 iu, vfpu)
CASE_LIST := \
      ISA_AMO \
      smoke_bus \
      ...
      sleep \
      iu \
      vfpu \
```

### 2. 新增 SEQ_LIST 变量

**位置**: Line 36

```makefile
SEQ_LIST := iu vfpu
```

**作用**: 
- 标记需要顺序编译的测试用例
- 这些用例的每个 .v 文件需要独立编译、独立运行
- 在 runcase 流程中触发循环处理逻辑

### 3. 新增 iu_build 和 vfpu_build targets

**位置**: Lines 144-155

```makefile
# iu_build - 整数单元测试构建
iu_build:
	@cp ./tests/cases/c910/iu/* ./work
	@find ./tests/lib/ -maxdepth 1 -type f -exec cp {} ./work/ \; 
	@cd ./work && make -s clean && make -s all CPU_ARCH_FLAG_0=c910 ENDIAN_MODE=little-endian CASENAME=iu FILE=tb_ct_iu_alu >& iu_build.case.log

# vfpu_build - 向量浮点单元测试构建
vfpu_build:
	@cp ./tests/cases/c910/vfpu/*.s ./work/$$(basename $(S_FILE) .v).s
	@cp ./tests/cases/c910/vfpu/$$(basename $(S_FILE)) ./work
	@find ./tests/lib/ -maxdepth 1 -type f -exec cp {} ./work/ \; 
	@cd ./work && make -s clean && make -s all CPU_ARCH_FLAG_0=c910 ENDIAN_MODE=little-endian CASENAME=vfpu FILE=$$(basename $(S_FILE) .v) >& vfpu_build.case.log
	@echo "end $$(basename $(S_FILE) .v)"
```

**关键技术点**:
- `$$(basename $(S_FILE) .v)` - 从 S_FILE 提取不带扩展的文件名
- vfpu 使用 S_FILE 参数指定具体的 .v 测试文件

### 4. 动态文件列表搜索 (vfpu)

**位置**: Lines 174-179

```makefile
ifeq ($(CASE), vfpu)
SIM_DIR := ../tests/cases/c910/vfpu/
SEARCH_DIR := ./tests/cases/c910/vfpu/
SIM_FILELIST := $(addprefix $(SIM_DIR)/, $(notdir $(wildcard $(SEARCH_DIR)/*.v)))
$(info SIM_FILELIST = $(SIM_FILELIST))
endif
```

**原理**:
```
┌─────────────────────────────────────────────────────────────┐
│                    文件列表生成流程                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  SEARCH_DIR := ./tests/cases/c910/vfpu/                      │
│                 ↑                                            │
│            wildcard 能访问的路径                              │
│                                                              │
│  $(wildcard $(SEARCH_DIR)/*.v)                               │
│      ↓                                                       │
│  [./tests/.../tb_ct_vfpu_top.v, ./tests/.../tb_ct_vfpu_dp.v] │
│                                                              │
│  $(notdir ...)                                               │
│      ↓                                                       │
│  [tb_ct_vfpu_top.v, tb_ct_vfpu_dp.v, ...]                    │
│                                                              │
│  $(addprefix $(SIM_DIR)/, ...)                               │
│      ↓                                                       │
│  [../tests/.../tb_ct_vfpu_top.v, ../tests/.../tb_ct_vfpu_dp.v]│
│                 ↑                                            │
│            VCS 编译需要的相对路径                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**为什么需要两个路径？**
- `SEARCH_DIR` (./tests...): Makefile wildcard 函数能正常访问
- `SIM_DIR` (../tests...): VCS 从 work 目录编译时需要的相对路径

---

## Makefile 更新详情

### 1. 默认仿真器变更

**位置**: Lines 22-26

```makefile
# 原版
SIM = iverilog
DUMP = off

# 更新版
#SIM = iverilog    # 注释掉，保留作为备选
SIM = vcs          # 默认使用 VCS
DUMP = on          # 默认开启波形 dump
```

### 2. 新增 S_FILE 和 SIM_FILELIST_copy

**位置**: Lines 73-74

```makefile
S_FILE ?=                      # 可选参数，指定单个 .v 文件
SIM_FILELIST_copy:= $(SIM_FILELIST)   # 保存原始列表用于循环
```

**作用**:
- `S_FILE`: 在 SEQ_LIST 模式下，传递当前处理的 .v 文件名
- `SIM_FILELIST_copy`: 因为 SIM_FILELIST 会被追加 RTL 文件列表，需要保留原始测试文件列表

### 3. VCS 编译选项增强

**位置**: Line 93

```makefile
# 原版
SIMULATOR_OPT := -sverilog -full64 -kdb -lca -debug_access +nospecify +notimingchecks +lint=TFIPC-L

# 更新版 (新增 -debug_all)
SIMULATOR_OPT := -sverilog -full64 -kdb -lca -debug_access +nospecify +notimingchecks +lint=TFIPC-L -debug_all
```

### 4. compile 目标改造

**位置**: Lines 137-161

```makefile
compile:
	@echo "  [THead-smart] Compiling smart now ... "
	@echo "  [THead-smart] SIM = $(SIM)"
ifeq ($(SIM), vcs)
	@make -s cleansim
ifeq ($(findstring $(CASE), $(SEQ_LIST)), $(CASE))
	@cd ./work && vcs $(SIMULATOR_OPT) $(TIMESCALE) $(SIMULATOR_DEF) $(S_FILE) $(SIM_DUMP) $(SIMULATOR_LOG) $(SIMULATOR_POWER_OPT)
else
	@cd ./work && vcs $(SIMULATOR_OPT) $(TIMESCALE) $(SIMULATOR_DEF) $(SIM_FILELIST) $(SIM_DUMP) $(SIMULATOR_LOG) $(SIMULATOR_POWER_OPT)
endif
```

**关键变更**:
- SEQ_LIST 用例使用 `$(S_FILE)` 单文件编译
- 普通用例使用 `$(SIM_FILELIST)` 批量编译

### 5. buildcase 目标改造

**位置**: Lines 181-200

```makefile
buildcase: tool-chain-chk
ifeq ($(CASE),)
	$(error Please specify CASE=xxx...)
endif
ifeq ($(findstring $(CASE), $(CASE_LIST)), $(CASE))
	@make -s cleancase
ifeq ($(findstring $(CASE), $(SEQ_LIST)), $(CASE))
	@echo $(S_FILE)
	@make -s $(CASE)_build S_FILE=$(S_FILE)    # 传递 S_FILE 参数
else
	@make -s $(CASE)_build
endif
```

### 6. runcase 目标重构 (核心变更)

**位置**: Lines 213-241

```makefile
runcase:
	@make cleanVerilator
ifeq ($(CASE),)
	$(error Please specify CASE=xxx...)
endif
ifeq ($(findstring $(CASE), $(CASE_LIST)), $(CASE))
ifeq ($(findstring $(CASE), $(SEQ_LIST)), $(CASE))
	# SEQ_LIST 用例: 循环处理每个文件
	@cd ./work && rm -rf *_seq.log && cd ..
	@for file in $(SIM_FILELIST_copy); do \
		echo "compile $${file}"; \
		make -s compile S_FILE="$${file} -f ../logical/filelists/sim.fl"; \
		echo "build $${file}"; \
		make -s buildcase CASE=$(CASE) S_FILE=$${file}; \
		make -s run_sim; \
	done
else
	# 普通用例: 单次执行
	@make -s compile
	@make -s buildcase CASE=$(CASE)
	@make -s run_sim
endif
```

### 7. 新增 run_sim 目标

**位置**: Lines 243-257

```makefile
run_sim:
ifeq ($(SIM), vcs)
	@cd ./work && ./simv -l run.vcs.log $(SIMV_POWER_OPT) && cat run.vcs.log >> $(CASE)_seq.log
else
    ifeq ($(SIM), nc)
		cd ./work && irun -R -l run.irun.log
    else
        ifeq ($(SIM), verilator)
		make buildVerilator
		cd ./work && obj_dir/Vtop
        else
	    cd ./work && vvp xuantie_core.vvp
        endif
    endif
endif
```

**作用**: 
- 独立运行仿真可执行文件
- SEQ_LIST 模式下将结果追加到 `*_seq.log`

### 8. 新增 verdi 目标

**位置**: Lines 270-271

```makefile
verdi:
	@cd ./work && verdi -sv -f ../logical/filelists/sim.fl -ssy -ssv -top tb -ssf novas.fsdb &
```

---

## 核心流程解析

### 普通用例执行流程 (非 SEQ_LIST)

```
make runcase CASE=coremark
         │
         ▼
    ┌─────────┐
    │ compile │ ─── 编译所有 RTL + 测试bench
    └─────────┘
         │
         ▼
    ┌──────────┐
    │ buildcase│ ─── 编译测试程序生成 .hex
    └──────────┘
         │
         ▼
    ┌─────────┐
    │ run_sim │ ─── 执行仿真
    └─────────┘
         │
         ▼
    结果输出: run.vcs.log
```

### SEQ_LIST 用例执行流程 (vfpu/iu)

```
make runcase CASE=vfpu
         │
         ▼
    SIM_FILELIST_copy = [tb_vfpu_top.v, tb_vfpu_dp.v, ...]
         │
         ▼
    ┌───────────────────────────────────────┐
    │ for file in $(SIM_FILELIST_copy):     │
    │                                        │
    │   ┌─────────┐                          │
    │   │ compile │ S_FILE=tb_vfpu_top.v    │
    │   └─────────┘                          │
    │        │                               │
    │   ┌──────────┐                         │
    │   │ buildcase│ S_FILE=tb_vfpu_top.v   │
    │   └──────────┘                         │
    │        │                               │
    │   ┌─────────┐                          │
    │   │ run_sim │                          │
    │   └─────────┘                          │
    │        │                               │
    │   结果追加到 vfpu_seq.log              │
    │                                        │
    │   ↓ 下一个文件                          │
    │                                        │
    │   ... 重复上述流程                      │
    │                                        │
    └───────────────────────────────────────┘
         │
         ▼
    最终输出: vfpu_seq.log (包含所有测试结果)
```

---

## 新增功能使用指南

### 1. 运行单个 vfpu 测试文件

```bash
# 运行所有 vfpu 测试
make runcase CASE=vfpu

# 仅运行特定文件 (需在 vfpu_build 中单独处理)
make runcase CASE=vfpu S_FILE=tb_ct_vfpu_rbus.v
```

### 2. 查看 vfpu 测试文件列表

```bash
make runcase CASE=vfpu
# 输出:
# SIM_FILELIST = ../tests/cases/c910/vfpu/tb_ct_vfpu_top.v ...
```

### 3. 查看所有可用测试用例

```bash
make showcase
# 输出:
#   Case lists:
#     ISA_AMO
#     smoke_bus
#     ...
#     iu
#     vfpu
```

### 4. 使用不同仿真器

```bash
# VCS (默认)
make runcase CASE=vfpu

# iverilog
make runcase CASE=vfpu SIM=iverilog

# Verilator
make runcase CASE=vfpu SIM=verilator THREADS=8
```

---

## Makefile 技术要点

### 1. wildcard 函数路径问题

**问题**: `$(wildcard ../tests/*.v)` 可能返回空

**原因**: Makefile 的 wildcard 对上级目录 `../` 相对路径有时失效

**解决方案**:
```makefile
SEARCH_DIR := ./tests/cases/c910/vfpu/
SIM_DIR := ../tests/cases/c910/vfpu/
SIM_FILELIST := $(addprefix $(SIM_DIR)/, $(notdir $(wildcard $(SEARCH_DIR)/*.v)))
```

### 2. 变量复制用于循环

**问题**: `SIM_FILELIST` 在 compile 中被追加 RTL 文件列表

**解决方案**:
```makefile
SIM_FILELIST_copy:= $(SIM_FILELIST)   # 在追加前保存副本
```

### 3. findstring 函数用于列表检查

```makefile
ifeq ($(findstring $(CASE), $(SEQ_LIST)), $(CASE))
    # CASE 在 SEQ_LIST 中
endif
```

### 4. basename 函数提取文件名

```makefile
$(basename $(S_FILE) .v)  # tb_ct_vfpu_top.v -> tb_ct_vfpu_top
```

### 5. shell 循环中传递 Makefile 变量

```makefile
@for file in $(SIM_FILELIST_copy); do \
    make -s compile S_FILE="$${file}"; \   # $${file} 是 shell 变量
done
```

**注意**:
- `$(VAR)` - Makefile 变量
- `$$VAR` 或 `$${VAR}` - shell 变量（双 $$ 转义）

---

## 文件对比总结

### smart_cfg.mk

| 位置 | 原版 | 更新版 | 变更说明 |
|------|------|--------|----------|
| Line 32-33 | 无 iu/vfpu | 新增 iu vfpu | CASE_LIST 扩展 |
| Line 36 | 无 | SEQ_LIST := iu vfpu | 新增顺序编译标记 |
| Line 144-147 | 无 | iu_build | 新增整数单元构建 |
| Line 150-155 | 无 | vfpu_build | 新增向量浮点构建 |
| Line 171-172 | 无 | ifeq ($(CASE), iu) | iu 文件列表 |
| Line 174-179 | 无 | ifeq ($(CASE), vfpu) | vfpu 动态文件搜索 |

### Makefile

| 位置 | 原版 | 更新版 | 变更说明 |
|------|------|--------|----------|
| Line 22 | SIM = iverilog | SIM = vcs | 默认仿真器变更 |
| Line 23 | DUMP = off | DUMP = on | 默认开启波形 |
| Line 73 | 无 | S_FILE ?= | 新增单文件参数 |
| Line 74 | 无 | SIM_FILELIST_copy | 保存文件列表副本 |
| Line 93 | 无 -debug_all | 新增 -debug_all | VCS 调试增强 |
| Line 142-143 | 无分支 | SEQ_LIST 分支 | 顺序编译支持 |
| Line 190-192 | 无分支 | SEQ_LIST 分支 | buildcase 支持 |
| Line 213-241 | 简单流程 | 循环流程 | runcase 重构 |
| Line 243-257 | 无 | run_sim target | 独立仿真执行 |
| Line 270-271 | 无 | verdi target | 新增波形查看 |

---

**文档生成时间**: 2026-04-17  
**项目**: OpenC910 C910 仿真系统  
**更新内容**: vfpu/iu 批量测试支持
XLEN ?= 32
DESTDIR ?= .
RTL_DIR = ../../hw/rtl
DPI_DIR = ../../hw/dpi
AFU_DIR = $(RTL_DIR)/afu/opae
SCRIPT_DIR = ../../hw/scripts
THIRD_PARTY_DIR = ../../third_party

CXXFLAGS += -std=c++11 -Wall -Wextra -Wfatal-errors -Wno-array-bounds
CXXFLAGS += -fPIC -Wno-maybe-uninitialized
CXXFLAGS += -I.. -I../../../hw -I../../common -I$(abspath $(DESTDIR))
CXXFLAGS += -I../$(THIRD_PARTY_DIR)/softfloat/source/include
CXXFLAGS += -I../$(THIRD_PARTY_DIR)
CXXFLAGS += -DXLEN_$(XLEN)

LDFLAGS += -shared ../$(THIRD_PARTY_DIR)/softfloat/build/Linux-x86_64-GCC/softfloat.a
LDFLAGS += -L../$(THIRD_PARTY_DIR)/ramulator -lramulator -pthread

# control RTL debug tracing states
DBG_TRACE_FLAGS += -DDBG_TRACE_CORE_PIPELINE  
DBG_TRACE_FLAGS += -DDBG_TRACE_CORE_ICACHE
DBG_TRACE_FLAGS += -DDBG_TRACE_CORE_DCACHE
DBG_TRACE_FLAGS += -DDBG_TRACE_CORE_MEM
DBG_TRACE_FLAGS += -DDBG_TRACE_CACHE_BANK 
DBG_TRACE_FLAGS += -DDBG_TRACE_CACHE_MSHR
DBG_TRACE_FLAGS += -DDBG_TRACE_CACHE_TAG
DBG_TRACE_FLAGS += -DDBG_TRACE_CACHE_DATA
DBG_TRACE_FLAGS += -DDBG_TRACE_AFU
DBG_TRACE_FLAGS += -DDBG_TRACE_SCOPE
DBG_TRACE_FLAGS += -DDBG_TRACE_GBAR

# Control logic analyzer monitors
DBG_SCOPE_FLAGS += -DDBG_SCOPE_AFU
DBG_SCOPE_FLAGS += -DDBG_SCOPE_ISSUE
DBG_SCOPE_FLAGS += -DDBG_SCOPE_FETCH
DBG_SCOPE_FLAGS += -DDBG_SCOPE_LSU
DBG_SCOPE_FLAGS += -DDBG_SCOPE_MSCHED

# AFU parameters
CONFIGS += -DPLATFORM_PROVIDES_LOCAL_MEMORY
ifeq (,$(findstring PLATFORM_PARAM_LOCAL_MEMORY_BANKS,$(CONFIGS)))
	CONFIGS += -DPLATFORM_PARAM_LOCAL_MEMORY_BANKS=2
endif
ifeq (,$(findstring PLATFORM_PARAM_LOCAL_MEMORY_ADDR_WIDTH,$(CONFIGS)))
	CONFIGS += -DPLATFORM_PARAM_LOCAL_MEMORY_ADDR_WIDTH=26
endif
ifeq (,$(findstring PLATFORM_PARAM_LOCAL_MEMORY_DATA_WIDTH,$(CONFIGS)))
	CONFIGS += -DPLATFORM_PARAM_LOCAL_MEMORY_DATA_WIDTH=512
endif
ifeq (,$(findstring PLATFORM_PARAM_LOCAL_MEMORY_BURST_CNT_WIDTH,$(CONFIGS)))
	CONFIGS += -DPLATFORM_PARAM_LOCAL_MEMORY_BURST_CNT_WIDTH=4
endif

DBG_FLAGS += -DDEBUG_LEVEL=$(DEBUG) -DVCD_OUTPUT $(DBG_TRACE_FLAGS)

SRCS = ../common/util.cpp ../common/mem.cpp ../common/rvfloats.cpp
SRCS += $(DPI_DIR)/util_dpi.cpp $(DPI_DIR)/float_dpi.cpp
SRCS += fpga.cpp opae_sim.cpp

RTL_PKGS = $(AFU_DIR)/local_mem_cfg_pkg.sv $(AFU_DIR)/ccip/ccip_if_pkg.sv
RTL_PKGS += $(RTL_DIR)/VX_gpu_pkg.sv $(RTL_DIR)/fpu/VX_fpu_pkg.sv

FPU_INCLUDE = -I$(RTL_DIR)/fpu
ifneq (,$(findstring FPU_FPNEW,$(CONFIGS)))
	RTL_PKGS += $(THIRD_PARTY_DIR)/fpnew/src/fpnew_pkg.sv $(THIRD_PARTY_DIR)/fpnew/src/common_cells/src/cf_math_pkg $(THIRD_PARTY_DIR)/fpnew/src/fpu_div_sqrt_mvp/hdl/defs_div_sqrt_mvp.sv
	FPU_INCLUDE += -I$(THIRD_PARTY_DIR)/fpnew/src/common_cells/include -I$(THIRD_PARTY_DIR)/fpnew/src/common_cells/src -I$(THIRD_PARTY_DIR)/fpnew/src/fpu_div_sqrt_mvp/hdl -I$(THIRD_PARTY_DIR)/fpnew/src
endif
RTL_INCLUDE = -I$(RTL_DIR) -I$(DPI_DIR) -I$(RTL_DIR)/libs -I$(RTL_DIR)/interfaces -I$(RTL_DIR)/core -I$(RTL_DIR)/mem -I$(RTL_DIR)/cache $(FPU_INCLUDE)
RTL_INCLUDE += -I$(AFU_DIR) -I$(AFU_DIR)/ccip

TOP = vortex_afu_shim

VL_FLAGS += --language 1800-2009 --assert -Wall -Wpedantic
VL_FLAGS += -Wno-DECLFILENAME -Wno-REDEFMACRO
VL_FLAGS += --x-initial unique --x-assign unique
VL_FLAGS += -DSIMULATION
VL_FLAGS += -DXLEN_$(XLEN)
VL_FLAGS += $(CONFIGS)
VL_FLAGS += verilator.vlt
VL_FLAGS += $(RTL_INCLUDE)
VL_FLAGS += $(RTL_PKGS)
VL_FLAGS += $(DBG_SCOPE_FLAGS)

CXXFLAGS += $(CONFIGS)

# Enable Verilator multithreaded simulation
THREADS ?= $(shell python -c 'import multiprocessing as mp; print(mp.cpu_count())')
VL_FLAGS += -j $(THREADS)
#VL_FLAGS += --threads $(THREADS)

# Debugigng
ifdef DEBUG
	VL_FLAGS += --trace --trace-structs $(DBG_FLAGS)
	CXXFLAGS += -g -O0 $(DBG_FLAGS)
else    
	VL_FLAGS += -DNDEBUG
	CXXFLAGS += -O3 -DNDEBUG
endif

# Enable scope analyzer
ifdef SCOPE
	VL_FLAGS += -DSCOPE
	CXXFLAGS += -DSCOPE
	SCOPE_JSON = $(DESTDIR)/scope.json
endif

# Enable perf counters
ifdef PERF
	VL_FLAGS += -DPERF_ENABLE
	CXXFLAGS += -DPERF_ENABLE
endif

# use our OPAE shim
VL_FLAGS += -DNOPAE
CXXFLAGS += -DNOPAE

PROJECT = libopae-c-sim.so

all: $(PROJECT)

$(DESTDIR)/vortex.xml:
	verilator --xml-only -O0 $(VL_FLAGS) $(TOP) --xml-output $(DESTDIR)/vortex.xml

$(DESTDIR)/scope.json: $(DESTDIR)/vortex.xml
	$(SCRIPT_DIR)/scope.py $(DESTDIR)/vortex.xml -o $(DESTDIR)/scope.json

$(DESTDIR)/vortex_afu.h : $(AFU_DIR)/vortex_afu.vh
	$(SCRIPT_DIR)/gen_config.py -i $(AFU_DIR)/vortex_afu.vh -o $(DESTDIR)/vortex_afu.h

$(DESTDIR)/$(PROJECT): $(SRCS) $(DESTDIR)/vortex_afu.h $(SCOPE_JSON)
	verilator --build --exe -O3 $(VL_FLAGS) --cc $(TOP) --top-module $(TOP) $(SRCS) -CFLAGS '$(CXXFLAGS)' -LDFLAGS '$(LDFLAGS)' -o ../$(DESTDIR)/$(PROJECT)

clean:
	rm -rf obj_dir $(DESTDIR)/vortex.xml $(DESTDIR)/scope.json $(DESTDIR)/vortex_afu.h $(DESTDIR)/$(PROJECT)

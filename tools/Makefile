# SPDX-License-Identifier: Apache-2.0
# Copyright 2020 Western Digital Corporation or its affiliates.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
CONF_PARAMS = -set iccm_enable
export RV_ROOT := $(shell pwd)

# Check for RV_ROOT
ifeq (,$(wildcard ${RV_ROOT}/configs/veer.config))
$(error env var RV_ROOT does not point to a valid dir! Exiting!)
endif

TEST_CFLAGS = -g -O3 -funroll-all-loops
ABI = -mabi=ilp32 -march=rv32imc_zicsr_zifencei

# Allow snapshot override
target = default
snapshot = $(target)

# Allow tool override
VEER_CONFIG = ${RV_ROOT}/configs/veer.config
VERILATOR = verilator
GCC_PREFIX = riscv64-unknown-elf
BUILD_DIR = snapshots/${snapshot}
TBDIR = ${RV_ROOT}/testbench

# Define default test name
TEST = hello_world

# Define default test directory
TEST_DIR = $(TBDIR)/ctests
HEX_DIR = $(TBDIR)/hex
ifneq (,$(wildcard $(TBDIR)/asm/$(TEST)))
        TEST_DIR = $(TBDIR)/asm/$(TEST)
endif

ifdef debug
 DEBUG_PLUS = +dumpon
 VERILATOR_DEBUG = --trace
endif

# provide specific link file
ifeq (,$(wildcard $(TEST_DIR)/$(TEST).ld))
	LINK = $(TBDIR)/link.ld
else
	LINK = $(TEST_DIR)/$(TEST).ld
endif

OFILES = $(TEST).o

-include $(TEST_DIR)/$(TEST).mki

VPATH = $(TEST_DIR) $(BUILD_DIR) $(TBDIR)
TBFILES = $(TBDIR)/tb_top.sv $(TBDIR)/axi_sif.sv

defines = $(BUILD_DIR)/common_defines.vh ${RV_ROOT}/design/include/veer_types.sv
includes = -I${RV_ROOT}/design/include -I${RV_ROOT}/design/lib  -I${BUILD_DIR}

# CFLAGS for verilator generated Makefiles. Without -std=c++11 it complains for `auto` variables
CFLAGS += "-std=c++14"
# Optimization for better performance; alternative is nothing for slower runtime (faster compiles)
# -O2 for faster runtime (slower compiles), or -O for balance.
VERILATOR_MAKE_FLAGS = OPT_FAST="-Os"

# Targets
all: clean verilator

clean:
	rm -rf *.log *.s *.hex *.dis *.tbl irun* vcs* simv* *.map snapshots \
	verilator* *.exe obj* *.o ucli.key vc_hdrs.h csrc *.csv \
	work dataset.asdb  library.cfg

# If define files do not exist, then run veer.config.
${BUILD_DIR}/defines.h :
	BUILD_PATH=${BUILD_DIR} ${VEER_CONFIG} -target=$(target) $(CONF_PARAMS)

##################### Verilog Builds #####################################

verilator-build: ${TBFILES} ${BUILD_DIR}/defines.h test_tb_top.cpp
	echo '`undef ASSERT_ON' >> ${BUILD_DIR}/common_defines.vh
	$(VERILATOR) --cc -CFLAGS ${CFLAGS} $(defines) $(includes) \
                -Wno-UNOPTFLAT \
                -I${RV_ROOT}/testbench \
		-f ${RV_ROOT}/testbench/flist \
                ${TBFILES} \
                --top-module tb_top -exe test_tb_top.cpp --autoflush $(VERILATOR_DEBUG)
	cp ${RV_ROOT}/testbench/test_tb_top.cpp obj_dir
	$(MAKE) -j -C obj_dir/ -f Vtb_top.mk $(VERILATOR_MAKE_FLAGS)
	touch verilator-build

##################### Simulation Runs #####################################

verilator: program.hex verilator-build
	./obj_dir/Vtb_top ${DEBUG_PLUS}


##################### Test Build #####################################


ifeq ($(shell which $(GCC_PREFIX)-gcc 2> /dev/null),)
program.hex: ${BUILD_DIR}/defines.h
	@echo " !!! No $(GCC_PREFIX)-gcc in path, using canned hex files !!"
	cp ${HEX_DIR}/$(TEST).hex program.hex
else
ifneq (,$(wildcard $(TEST_DIR)/$(TEST).makefile))
program.hex:
	$(MAKE) -f $(TEST_DIR)/$(TEST).makefile
else
program.hex: $(OFILES) $(LINK)
	@echo Building $(TEST)
	$(GCC_PREFIX)-gcc $(ABI) -Wl,-Map=$(TEST).map -lgcc -T$(LINK) -o $(TEST).exe $(OFILES) -nostartfiles  $(TEST_LIBS)
	$(GCC_PREFIX)-objcopy -O verilog $(TEST).exe program.hex
	$(GCC_PREFIX)-objdump -S  $(TEST).exe > $(TEST).dis
	@echo Completed building $(TEST)

%.o : %.s ${BUILD_DIR}/defines.h
	$(GCC_PREFIX)-cpp -I${BUILD_DIR}  $<  > $*.cpp.s
	$(GCC_PREFIX)-as $(ABI) $*.cpp.s -o $@


%.o : %.c ${BUILD_DIR}/defines.h
	$(GCC_PREFIX)-gcc -I${BUILD_DIR} ${TEST_CFLAGS} ${ABI} -nostdlib -c $< -o $@
endif
endif

help:
	@echo Make sure the environment variable RV_ROOT is set.
	@echo Possible targets: verilator help clean all verilator-build program.hex

.PHONY: help clean verilator

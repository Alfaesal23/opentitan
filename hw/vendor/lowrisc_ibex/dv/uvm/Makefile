# Copyright lowRISC contributors.
# Licensed under the Apache License, Version 2.0, see LICENSE for details.
# SPDX-License-Identifier: Apache-2.0

DV_DIR              := $(shell dirname $(realpath $(lastword $(MAKEFILE_LIST))))
GEN_DIR             := $(realpath ${DV_DIR}/../../vendor/google_riscv-dv)
TOOLCHAIN           := ${RISCV_TOOLCHAIN}
OUT                 := "${DV_DIR}/out"
# Run time options for the instruction generator
GEN_OPTS            :=
# Run time options for ibex RTL simulation
SIM_OPTS            :=
# Enable waveform dumping
WAVES               := 1
# Enable coverage dump
COV                 := 0
# RTL simulator
SIMULATOR           := "vcs"
# ISS (spike, ovpsim)
ISS                 := "spike"
# ISA
ISA                 := "rv32imc"
# Test name (default: full regression)
TEST                := "all"
# Seed for instruction generator and RTL simulation
SEED                := -1
# Verbose logging
VERBOSE             :=
# Number of iterations for each test, assign a non-zero value to override the
# iteration count in the test list
ITERATIONS          := 0
# LSF CMD
LSF_CMD             :=
# Generator timeout limit in seconds
TIMEOUT             := 1800
# Privileged CSR YAML description file
CSR_FILE            := ${DV_DIR}/riscv_dv_extension/csr_description.yaml
# Pass/fail signature address at the end of test
SIGNATURE_ADDR      := 8ffffffc

SHELL=/bin/bash

export PRJ_DIR:= $(realpath ${DV_DIR}/../../..)

.PHONY: rtl_sim clean gcc_compile iss_sim

all: clean gen gcc_compile iss_sim compile rtl_sim post_compare

clean:
	rm -rf ${OUT}

# Common options for all targets
COMMON_OPTS:=--seed=${SEED} \
             --test=${TEST} \
             --testlist=${DV_DIR}/riscv_dv_extension/testlist.yaml \
             --iterations=${ITERATIONS}

ifeq ($(VERBOSE), 1)
	COMMON_OPTS+=--verbose
endif

# Options used for privileged CSR test generation
CSR_OPTS=--csr_yaml=${CSR_FILE} \
         --isa=${ISA} \
         --end_signature_addr=0x${SIGNATURE_ADDR}

# Generate random instructions
.SILENT gen:
	mkdir -p ${OUT}
	cd ${GEN_DIR}; \
	python3 ./run.py \
     --output=${OUT}/instr_gen ${GEN_OPTS} \
     --steps=gen \
     --gen_timeout=${TIMEOUT} \
     --lsf_cmd="${LSF_CMD}" \
     --core_setting_dir=${DV_DIR}/riscv_dv_extension \
     --user_extension_dir=${DV_DIR}/riscv_dv_extension \
     --simulator=${SIMULATOR} \
     ${COMMON_OPTS} \
     ${CSR_OPTS} \
     --sim_opts="+uvm_set_type_override=riscv_asm_program_gen,ibex_asm_program_gen \
                 +signature_addr=${SIGNATURE_ADDR}";

# Compile the generated assmebly programs to ELF/BIN
gcc_compile:
	cd ${GEN_DIR}; \
	python3 ./run.py \
     --o=${OUT}/instr_gen ${GEN_OPTS} \
     --steps=gcc_compile \
     ${COMMON_OPTS} \
     --gcc_opts=-mno-strict-align \
     --isa=${ISA} \
     --mabi=ilp32

# ISS simulation
iss_sim:
	cd ${GEN_DIR}; \
	python3 ./run.py \
     --o=${OUT}/instr_gen ${GEN_OPTS} \
     --steps=iss_sim \
     ${COMMON_OPTS} \
     --iss=${ISS} \
     --isa=${ISA} \

# Compile ibex core TB
compile:
	mkdir -p ${OUT}/rtl_sim
	python3 ./sim.py \
     --o=${OUT} \
     --steps=compile \
     ${COMMON_OPTS} \
     --simulator=${SIMULATOR} \
     --en_cov=${COV} \
     --en_wave=${WAVES} \

# Run ibex RTL simulation with random instructions
rtl_sim:
	mkdir -p ${OUT}/rtl_sim
	python3 ./sim.py \
     --o=${OUT} \
     --steps=sim \
     ${COMMON_OPTS} \
     --simulator=${SIMULATOR} \
     --en_cov ${COV} \
     --en_wave ${WAVES} \
     --lsf_cmd="${LSF_CMD}" \
     --sim_opts="+signature_addr=0x${SIGNATURE_ADDR}" \
     ${SIM_OPTS}

# Compare the regression result between ISS and RTL sim
post_compare:
	rm -rf ${OUT}/regr.log
	python3 ./sim.py \
     --o=${OUT} \
     --steps=compare \
     ${COMMON_OPTS} \
     --simulator=${SIMULATOR} \
     --iss=${ISS}


# Generate functional coverage
fcov:
	cd ${GEN_DIR}; \
	python3 ./cov.py --dir ${OUT}/instr_gen/spike_sim -o ${OUT}/fcov

# Load verdi to review coverage
cov:
	cd ${OUT}/rtl_sim; verdi -cov -covdir test.vdb &

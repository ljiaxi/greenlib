# enabling SystemC2.1 automatically changes 
# includes and compiler flags in Makefile.defs
#SYSTEMC21V1 = yes

CC     = g++
#OPT    = -O3 -funroll-loops -mtune=pentium4
OPT  = -g -Wall -lpthread -D_REENTRANT
#OTHER  = -Wno-deprecated
CFLAGS = $(OPT) $(OTHER)
# CFLAGS = $(DEBUG) $(OTHER)
# EXTRA_LIBS = -lboost_program_options

MODULE = testbench_tool_api
SRCS = testbench_tool_api.cpp testip4.cpp testtool2.cpp
OBJS = $(SRCS:.cpp=.o)

include ../Makefile.gspm
include ../Makefile.defs

all: $(EXE)


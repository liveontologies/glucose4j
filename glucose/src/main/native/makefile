#-----------------------------------------------------------------------#
#- GLOBAL DEFS ---------------------------------------------------------#
#-----------------------------------------------------------------------#

# Keep this as generic as possible.

NAME=${library.name}
VERSION=${library.version}
ARCHIVE=${library.archive.name}
URL=${library.download.url}

#-----------------------------------------------------------------------#
# Solver signatures have to be both valid file names and C symbols.
# Since Picosat uses a dash '-' for the package name, we have to
# differentiate between with (directory / package name) and without
# dash (the signature).

SIG=$(NAME)$(VERSION)
DIR=$(ARCHIVE)
TARGET=dlib

#-----------------------------------------------------------------------#

export CXX 		?= g++
export AR		+= -rcsv
export MAKE		?= make
export CFLAGS	?= -std=c++11 -Wall -DNDEBUG -O3 -fPIC
CFLAGS += -DVERSION=\"$(VERSION)\"
# zlib is not provided for mingw32
ZLIB_VERSION	?= 1.3.1

#-----------------------------------------------------------------------#
#- REQUIRED TOP RULES --------------------------------------------------#
#-----------------------------------------------------------------------#

all: $(TARGET)

clean:
	rm -rf $(DIR) zlib-$(ZLIB_VERSION)
	rm -f *.o *.a *.dylib *.so *.dll

#-----------------------------------------------------------------------#
#- INVISIBLE INTERNAL SUB RULES ----------------------------------------#
#-----------------------------------------------------------------------#

# enable verbatim output
ifeq ($(VERB),)
VB=@
else
VB=
endif

# cross-compilation
ifeq ($(CHOST),)
else
	CROSS_PREFIX = $(CHOST)-
	CXX := $(CROSS_PREFIX)$(CXX)
	AR := $(CROSS_PREFIX)$(AR)
	UNAME = $(shell echo "$(CHOST)" | sed -e 's/^[^-]*-\([^-]*\)\s*$$/\1/' -e 's/^[^-]*-[^-]*-\([^-]*\)\s*$$/\1/' -e 's/^[^-]*-[^-]*-\([^-]*\)-.*$$/\1/')
endif

# determine operating system name
UNAME ?= $(shell uname -s)

# the location of this makefile (independent of the current directory)
MROOT := $(dir $(abspath $(lastword $(MAKEFILE_LIST))))

ifeq ($(UNAME), Darwin)
  DLIB = lib$(NAME).dylib
  # build universal "fat" binaries
  export CFLAGS += -arch x86_64 -arch arm64
  AR = libtool -static -o
else ifeq ($(UNAME), Linux)
  DLIB = lib$(NAME).so
else ifeq ($(UNAME), mingw32)
  DLIB = $(NAME).dll  
  export CFLAGS += -I$(MROOT)/zlib-$(ZLIB_VERSION) -L$(MROOT)/zlib-$(ZLIB_VERSION)
  # MinGW does not ship with zlib
  LIBS = zlib-$(ZLIB_VERSION)  
else
  $(error Platform not supported: $(UNAME))
endif 

STLIB=$(DIR)/simp/lib_release.a
GLUE=$(DIR)/core/ipasir$(NAME)glue

dlib: $(DLIB)

# shared library for Mac OS
lib$(NAME).dylib: $(STLIB)
	$(VB) $(CXX) $(CFLAGS) -g -I. -o $@ -dynamiclib -install_name $@ $(GLUE).or $^

# shared library for Linux
lib$(NAME).so: $(STLIB)
	$(VB) $(CXX) $(CFLAGS) -g -I. -o $@ -shared -Wl,-soname,$@ $(GLUE).or $^

# shared library for MinGW
$(NAME).dll: $(STLIB)	
	$(VB) $(CXX) $(CFLAGS) -static-libgcc -static-libstdc++ -g -I. -o $@ -shared -Wl,--subsystem,windows $(GLUE).or $^

# static solver library
$(STLIB): $(DIR) $(LIBS)	
	@#
	@# configure and build library
	@#
	$(VB) cd $(DIR)/simp; $(MAKE) VERBOSE=1 libr

$(DIR): $(ARCHIVE).tgz $(DIR).patch
	@#
	@# extract library
	@#
	tar xf $<
	@# apply patch for set_terminate and learn_callback
	@#
	$(VB) patch -p0 < $(DIR).patch
	@#
	@# apply patch for mingw
	@#
ifeq ($(UNAME), mingw32)
	$(VB) patch -p0 < $(DIR).mingw.patch
endif
	@#
	@# patch verbose output
	@#
	$(VB) sed -i.bak -e 's,"c ,"c [${SIG}] ,' $(DIR)/core/Solver.cc
	$(VB) sed -i.bak -e 's,"c ,"c [${SIG}] ,' $(DIR)/simp/SimpSolver.cc
	$(VB) sed -i.bak -e 's,"c ,"c [${SIG}] ,' $(DIR)/simp/Main.cc
	@#
	@# patch archiver
	@#
	$(VB) sed -i.bak -e's/$$(AR) -rcsv/$$(AR)/' $(DIR)/mtl/template.mk
	@#
	@# copy glue code for ipasir
	@#
	$(VB) cp ipasir.h $(DIR)/core/
	$(VB) cp ipasir$(NAME)glue.cc $(GLUE).cc		

$(ARCHIVE).tgz:
	@#
	@# download library
	@#
	wget "$(URL)$@"

# zlib
zlib-$(ZLIB_VERSION): zlib-$(ZLIB_VERSION).tar.gz
	tar xzf $<
	@#
	@# patch makefile for mingw
	@#
	$(VB) sed -i.bak -e s/"PREFIX ="/"PREFIX = $(CROSS_PREFIX)"/ zlib-$(ZLIB_VERSION)/win32/Makefile.gcc
	$(VB) cd zlib-$(ZLIB_VERSION); $(MAKE) -f win32/Makefile.gcc
		

zlib-$(ZLIB_VERSION).tar.gz:
	$(VB) wget https://zlib.net/zlib-$(ZLIB_VERSION).tar.gz
	
#-----------------------------------------------------------------------#

.FORCE:
.PHONY: all clean

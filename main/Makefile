#
# Asterisk -- An open source telephony toolkit.
#
# Makefile to build main Asterisk binary
#
# Copyright (C) 1999-2006, Digium, Inc.
#
# Mark Spencer <markster@digium.com>
#
# This program is free software, distributed under the terms of
# the GNU General Public License
#

-include $(ASTTOPDIR)/menuselect.makeopts $(ASTTOPDIR)/menuselect.makedeps $(ASTTOPDIR)/makeopts

all: asterisk

include $(ASTTOPDIR)/Makefile.moddir_rules

MOD_SRC:=cdr.c cel.c config.c ccss.c dnsmgr.c dsp.c enum.c features.c http.c indications.c logger.c manager.c named_acl.c plc.c sounds.c udptl.c
# Allow deletion of built-in modules without needing to modify this source.
MOD_SRC:=$(wildcard $(MOD_SRC))
MOD_OBJS:=$(sort $(MOD_SRC:.c=.o))

# Must include the extra ast_expr2.c, ast_expr2f.c, in case they need to be regenerated (because to force regeneration, we delete them)
SRC:=$(wildcard *.c) ast_expr2.c ast_expr2f.c
ifeq ($(AST_ASTERISKSSL),yes)
SRC:=$(filter-out libasteriskssl.c,$(SRC))
endif
ifeq ($(PJPROJECT_BUNDLED),yes)
SRC:=$(filter-out libasteriskpj.c,$(SRC))
endif
OBJSFILTER:=$(MOD_OBJS) fskmodem_int.o fskmodem_float.o cygload.o buildinfo.o

SRC_CC:=$(filter-out $(addsuffix .cc,$(MENUSELECT_CHANNELSTORAGE)),$(wildcard *.cc))
SRC:=$(filter-out $(addsuffix .c,$(MENUSELECT_CHANNELSTORAGE)),$(SRC))

OBJS=$(filter-out $(OBJSFILTER),$(SRC:.c=.o) $(SRC_CC:.cc=.oo))

# we need to link in the objects statically, not as a library, because
# otherwise modules will not have them available if none of the static
# objects use it.
OBJS+=stdtime/localtime.o

ASTSSL_LIBS:=$(OPENSSL_LIB)
AST_LIBS+=$(BKTR_LIB)
AST_LIBS+=$(LIBXML2_LIB)
AST_LIBS+=$(LIBXSLT_LIB)
AST_LIBS+=$(SQLITE3_LIB)
AST_LIBS+=$(ASTSSL_LIBS)
AST_LIBS+=$(JANSSON_LIB)
AST_LIBS+=$(URIPARSER_LIB)
AST_LIBS+=$(UUID_LIB)
AST_LIBS+=$(CRYPT_LIB)
AST_LIBS+=$(AST_CLANG_BLOCKS_LIBS)
AST_LIBS+=$(RT_LIB)
AST_LIBS+=$(SYSTEMD_LIB)

ifneq ($(findstring $(OSARCH), linux-gnu uclinux linux-uclibc linux-musl kfreebsd-gnu),)
  AST_LIBS+=-ldl
  ifneq (x$(CAP_LIB),x)
    AST_LIBS+=$(CAP_LIB)
  endif
  AST_LIBS+=-lpthread -lm -lresolv
else
  AST_LIBS+=-lm
endif

ifneq ($(findstring BETTER_BACKTRACES,$(MENUSELECT_CFLAGS)),)
  AST_LIBS+=$(BFD_LIB)
endif

ifneq ($(findstring darwin,$(OSARCH)),)
  AST_LIBS+=-lresolv
  ASTLINK=-mmacosx-version-min=10.6 -Wl,-undefined,dynamic_lookup -force_flat_namespace
  ASTLINK+=/usr/lib/bundle1.o
else
# These are used for all but Darwin
  ASTLINK+=-Wl,--export-dynamic
  ifneq ($(findstring BSD,$(OSARCH)),)
    LDFLAGS+=-L/usr/local/lib
  endif
endif

ifeq ($(OSARCH),FreeBSD)
  # -V is understood by BSD Make, not by GNU make.
  BSDVERSION=$(shell make -V OSVERSION -f /usr/share/mk/bsd.port.subdir.mk)
  AST_LIBS+=$(shell if test $(BSDVERSION) -lt 502102 ; then echo "-lc_r"; else echo "-pthread"; fi)
  AST_LIBS+=-lcrypto
endif

ifneq ($(findstring $(OSARCH), mingw32 cygwin ),)
  AST_LIBS+=-lminires -ldl
  ASTLINK+=-shared -Wl,--out-implib,libasterisk.a
endif
ifeq ($(OSARCH),NetBSD)
  AST_LIBS+=-lpthread -lcrypto -lm -L/usr/pkg/lib
endif

ifeq ($(OSARCH),OpenBSD)
  AST_LIBS+=-lcrypto -lpthread -lm
endif

ifeq ($(OSARCH),SunOS)
  AST_LIBS+=-lpthread -ldl -lrt -lnsl -lsocket -lresolv
  ASTSSL_LIBS+=-L/opt/ssl/lib -L/usr/local/ssl/lib
  ASTLINK=
endif

ifneq ($(findstring USE_HOARD_ALLOCATOR,$(MENUSELECT_CFLAGS)),)
  ifneq ($(HOARD_LIB),)
    AST_LIBS+=$(HOARD_LIB)
  endif
endif

ifeq ($(GNU_LD),1)
ASTLINK+=-Wl,--version-script,asterisk.exports
  ifeq ($(HAVE_DYNAMIC_LIST),1)
  ASTLINK+=-Wl,--dynamic-list,asterisk.dynamics
  endif
endif

.PHONY: CHECK_SUBDIR
CHECK_SUBDIR:	# do nothing, just make sure that we recurse in the subdir/

ifneq ($(findstring REBUILD_PARSERS,$(MENUSELECT_CFLAGS)),)
ast_expr2.c ast_expr2.h: ast_expr2.y
else
ast_expr2.c ast_expr2.h:
endif
	$(ECHO_PREFIX) echo "   [BISON] $< -> $@"
	$(CMD_PREFIX) $(BISON) -o $@ -d --name-prefix=ast_yy ast_expr2.y

ifneq ($(findstring REBUILD_PARSERS,$(MENUSELECT_CFLAGS)),)
ast_expr2f.c: ast_expr2.fl
else
ast_expr2f.c:
endif
	$(ECHO_PREFIX) echo "   [FLEX] $< -> $@"
	$(CMD_PREFIX) echo "#define ASTMM_LIBC ASTMM_REDIRECT" > $@
	$(CMD_PREFIX) echo "#include \"asterisk.h\"" >> $@
	$(CMD_PREFIX) echo >> $@
	$(CMD_PREFIX) $(FLEX) -t ast_expr2.fl >> $@

ifneq ($(findstring ENABLE_UPLOADS,$(MENUSELECT_CFLAGS)),)
GMIMELDFLAGS+=$(GMIME_LIB)
GMIMECFLAGS+=$(GMIME_INCLUDE)
endif

# Alter CFLAGS for specific sources
stdtime/localtime.o: _ASTCFLAGS+=$(AST_NO_STRICT_OVERFLOW) -Wno-format-nonliteral

asterisk.o: _ASTCFLAGS+=$(LIBEDIT_INCLUDE)
ast_expr2f.o: _ASTCFLAGS+=-Wno-unused
astmm.o: _ASTCFLAGS+=$(call get_menuselect_cflags,MALLOC_DEBUG DEBUG_CHAOS)
astobj2.o astobj2_container.o astobj2_hash.o astobj2_rbtree.o: _ASTCFLAGS+=$(call get_menuselect_cflags,AO2_DEBUG)
backtrace.o: _ASTCFLAGS+=$(call get_menuselect_cflags,BETTER_BACKTRACES)
bucket.o: _ASTCFLAGS+=$(URIPARSER_INCLUDE)
cdr.o: _ASTCFLAGS+=$(AST_NO_FORMAT_TRUNCATION)
crypt.o: _ASTCFLAGS+=$(CRYPT_INCLUDE)
db.o: _ASTCFLAGS+=$(SQLITE3_INCLUDE)
dsp.o: _ASTCFLAGS+=$(call get_menuselect_cflags,RADIO_RELAX)
frame.o: _ASTCFLAGS+=$(call get_menuselect_cflags,MALLOC_DEBUG)
http.o: _ASTCFLAGS+=$(GMIMECFLAGS)
iostream.o: _ASTCFLAGS+=$(OPENSSL_INCLUDE)
json.o: _ASTCFLAGS+=$(JANSSON_INCLUDE)
lock.o: _ASTCFLAGS+=$(call get_menuselect_cflags,DETECT_DEADLOCKS)
options.o: _ASTCFLAGS+=$(call get_menuselect_cflags,REF_DEBUG)
sched.o: _ASTCFLAGS+=$(call get_menuselect_cflags,DEBUG_SCHEDULER DUMP_SCHEDULER)
tcptls.o: _ASTCFLAGS+=$(OPENSSL_INCLUDE) -Wno-deprecated-declarations
# since we're using open_memstream(), we need to release the buffer with
# the native free() function or we might get unexpected behavior.
test.o: _ASTCFLAGS+=-DASTMM_LIBC=ASTMM_IGNORE
uuid.o: _ASTCFLAGS+=$(UUID_INCLUDE)
stasis.o: _ASTCFLAGS+=$(call get_menuselect_cflags,AO2_DEBUG)
time.o: _ASTCFLAGS+=-D_XOPEN_SOURCE=700


OBJS:=$(sort $(OBJS))

ifneq ($(findstring $(OSARCH), mingw32 cygwin ),)
MAIN_TGT:=asterisk.dll
asterisk: cygload
	mv cygload.exe asterisk.exe

cygload: asterisk.dll
else
MAIN_TGT:=asterisk
endif

$(OBJS): _ASTCFLAGS+=-DAST_MODULE=\"core\" -DAST_IN_CORE
$(MOD_OBJS): _ASTCFLAGS+=$(call MOD_ASTCFLAGS,$*)

libasteriskssl.o: _ASTCFLAGS+=$(OPENSSL_INCLUDE)

ifeq ($(AST_ASTERISKSSL),yes)
# The ABI *version* of the asteriskssl library; don't change this unless there truly is a
# non-backwards-compatible ABI change in the library
ASTSSL_SO_VERSION=1

ASTSSL_LDLIBS=-L. -lasteriskssl

ifeq ($(findstring darwin,$(OSARCH)),) # not Darwin
ASTSSL_LIB:=libasteriskssl.so

$(ASTSSL_LIB).$(ASTSSL_SO_VERSION): _ASTLDFLAGS+=-Wl,-soname=$(ASTSSL_LIB).$(ASTSSL_SO_VERSION)
$(ASTSSL_LIB).$(ASTSSL_SO_VERSION): _ASTCFLAGS+=-fPIC -DAST_MODULE=\"asteriskssl\" -DAST_NOT_MODULE
$(ASTSSL_LIB).$(ASTSSL_SO_VERSION): LIBS+=$(ASTSSL_LIBS)
ifeq ($(GNU_LD),1)
    $(ASTSSL_LIB).$(ASTSSL_SO_VERSION): SO_SUPPRESS_SYMBOLS=-Wl,--version-script,libasteriskssl.exports,--warn-common
endif
$(ASTSSL_LIB).$(ASTSSL_SO_VERSION): SOLINK=$(DYLINK)

# These rules are duplicated from $(ASTTOPDIR)/Makefile.rules because the library name
# being built does not match the "%.so" pattern; there are also additional steps
# required to build a proper shared library (as opposed to the 'loadable module'
# type that are built by the standard rules)
$(ASTSSL_LIB).$(ASTSSL_SO_VERSION): libasteriskssl.o
ifeq ($(GNU_LD),1)
	$(CMD_PREFIX) $(ASTTOPDIR)/build_tools/make_linker_version_script libasteriskssl "$(LINKER_SYMBOL_PREFIX)" "$(ASTTOPDIR)"
endif
	$(ECHO_PREFIX) echo "   [LD] $^ -> $@"
	$(CMD_PREFIX) $(CC) -o $@ $(CC_LDFLAGS_SO) $^ $(CC_LIBS)

$(ASTSSL_LIB): $(ASTSSL_LIB).$(ASTSSL_SO_VERSION)
	$(ECHO_PREFIX) echo "   [LN] $< -> $@"
	$(LN) -sf $< $@ ;\

else # Darwin
ASTSSL_LIB:=libasteriskssl.dylib

# -install_name allows library to be found if installed somewhere other than
# /lib or /usr/lib
$(ASTSSL_LIB): _ASTLDFLAGS+=-dynamiclib -install_name $(ASTLIBDIR)/$(ASTSSL_LIB)
$(ASTSSL_LIB): _ASTCFLAGS+=-fPIC -DAST_MODULE=\"asteriskssl\" -DAST_NOT_MODULE
$(ASTSSL_LIB): LIBS+=$(ASTSSL_LIBS)
$(ASTSSL_LIB): SOLINK=$(DYLINK)

# Special rules for building a shared library (not a dynamically loadable module)
$(ASTSSL_LIB): libasteriskssl.o
	$(ECHO_PREFIX) echo "   [LD] $^ -> $@"
	$(CMD_PREFIX) $(CC) -o $@ $(CC_LDFLAGS_SO) $^ $(CC_LIBS)
endif

endif

libasteriskpj.o: _ASTCFLAGS+=$(PJPROJECT_INCLUDE)

ifeq ($(PJPROJECT_BUNDLED),yes)

ASTPJ_SO_VERSION=2
ASTPJ_LDLIBS=-L. -lasteriskpj

PJDIR=$(ASTTOPDIR)/$(PJPROJECT_DIR)/source
-include $(ASTTOPDIR)/$(PJPROJECT_DIR)/build.mak

PJPROJECT_LDLIBS := \
-Wl,--whole-archive \
$(PJSUA_LIB_LDLIB) \
$(PJSIP_UA_LDLIB) \
$(PJSIP_SIMPLE_LDLIB) \
$(PJSIP_LDLIB) \
$(PJNATH_LDLIB) \
$(PJMEDIA_CODEC_LDLIB) \
$(PJMEDIA_VIDEODEV_LDLIB) \
$(PJMEDIA_AUDIODEV_LDLIB) \
$(PJMEDIA_LDLIB) \
$(PJLIB_UTIL_LDLIB) \
$(PJLIB_LDLIB) \
-Wl,--no-whole-archive \
$(APP_THIRD_PARTY_LIBS) \
$(APP_THIRD_PARTY_EXT)

ifeq ($(findstring darwin,$(OSARCH)),) # not Darwin
ASTPJ_LIB:=libasteriskpj.so

libasteriskpj.exports: $(ASTTOPDIR)/$(PJPROJECT_DIR)/pjproject.symbols
	$(ECHO_PREFIX) echo "   [GENERATE] libasteriskpj.exports"
ifeq ($(GNU_LD),1)
	$(CMD_PREFIX) echo -e "{\nglobal:" > libasteriskpj.exports
	$(CMD_PREFIX) sed -r -e "s/.*/$(LINKER_SYMBOL_PREFIX)&;/" $(ASTTOPDIR)/$(PJPROJECT_DIR)/pjproject.symbols >> libasteriskpj.exports
	$(CMD_PREFIX) echo -e "$(LINKER_SYMBOL_PREFIX)ast_pj_init;\n" >> libasteriskpj.exports
	$(CMD_PREFIX) echo -e "local:\n*;\n};" >> libasteriskpj.exports
endif

$(ASTPJ_LIB).$(ASTPJ_SO_VERSION): _ASTLDFLAGS+=-Wl,-soname=$(ASTPJ_LIB).$(ASTPJ_SO_VERSION) $(PJ_LDFLAGS)
$(ASTPJ_LIB).$(ASTPJ_SO_VERSION): _ASTCFLAGS+=-fPIC -DAST_MODULE=\"asteriskpj\" -DAST_NOT_MODULE $(PJ_CFLAGS)
$(ASTPJ_LIB).$(ASTPJ_SO_VERSION): LIBS+=$(PJPROJECT_LDLIBS) $(OPENSSL_LIB) $(UUID_LIB) -lm -lpthread $(RT_LIB)
ifeq ($(GNU_LD),1)
    $(ASTPJ_LIB).$(ASTPJ_SO_VERSION): SO_SUPPRESS_SYMBOLS=-Wl,--version-script,libasteriskpj.exports,--warn-common
endif
$(ASTPJ_LIB).$(ASTPJ_SO_VERSION): SOLINK=$(DYLINK)

# These rules are duplicated from $(ASTTOPDIR)/Makefile.rules because the library name
# being built does not match the "%.so" pattern; there are also additional steps
# required to build a proper shared library (as opposed to the 'loadable module'
# type that are built by the standard rules)
$(ASTPJ_LIB).$(ASTPJ_SO_VERSION): libasteriskpj.o libasteriskpj.exports
	$(ECHO_PREFIX) echo "   [LD] $< -> $@"
	$(CMD_PREFIX) $(CC) -o $@ $(CC_LDFLAGS_SO) $< $(CC_LIBS)

$(ASTPJ_LIB): $(ASTPJ_LIB).$(ASTPJ_SO_VERSION)
	$(ECHO_PREFIX) echo "   [LN] $< -> $@"
	$(LN) -sf $< $@ ;\

else # Darwin
ASTPJ_LIB:=libasteriskpj.dylib

# -install_name allows library to be found if installed somewhere other than
# /lib or /usr/lib
$(ASTPJ_LIB): _ASTLDFLAGS+=-dynamiclib -install_name $(ASTLIBDIR)/$(ASTPJ_LIB) $(PJ_LDFLAGS)
$(ASTPJ_LIB): _ASTCFLAGS+=-fPIC -DAST_MODULE=\"asteriskpj\" $(PJ_CFLAGS) -DAST_NOT_MODULE
$(ASTPJ_LIB): LIBS+=$(PJPROJECT_LIB) $(OPENSSL_LIB) $(UUID_LIB) -lm -lpthread $(RT_LIB)
$(ASTPJ_LIB): SOLINK=$(DYLINK)

# Special rules for building a shared library (not a dynamically loadable module)
$(ASTPJ_LIB): libasteriskpj.o
	$(ECHO_PREFIX) echo "   [LD] $^ -> $@"
	$(CMD_PREFIX) $(CC) -o $@ $(CC_LDFLAGS_SO) $^ $(CC_LIBS)
endif

endif

$(MAIN_TGT): $(OBJS) $(MOD_OBJS) $(ASTSSL_LIB) $(ASTPJ_LIB)
	@$(CC) -c -o buildinfo.o $(_ASTCFLAGS) buildinfo.c $(ASTCFLAGS)
	$(ECHO_PREFIX) echo "   [LD] $(OBJS) $(MOD_OBJS) -> $@"
	$(CMD_PREFIX) $(CXX) -o $@ $(ASTLINK) $(_ASTLDFLAGS) $(ASTLDFLAGS) $(OBJS) $(MOD_OBJS) $(ASTSSL_LDLIBS) $(ASTPJ_LDLIBS) buildinfo.o $(AST_LIBS) $(GMIMELDFLAGS) $(LIBEDIT_LIB)

ifeq ($(GNU_LD),1)
$(MAIN_TGT): asterisk.exports
asterisk.exports: asterisk.exports.in
	$(CMD_PREFIX) $(ASTTOPDIR)/build_tools/make_linker_version_script asterisk $(LINKER_SYMBOL_PREFIX)
endif

bininstall:
	$(INSTALL) -m 755 $(MAIN_TGT) "$(DESTDIR)$(ASTSBINDIR)/"
ifeq ($(AST_ASTERISKSSL),yes)
ifeq ($(findstring darwin,$(OSARCH)),) # not Darwin
	$(INSTALL) -m 755 $(ASTSSL_LIB).$(ASTSSL_SO_VERSION) "$(DESTDIR)$(ASTLIBDIR)/"
	$(LN) -sf $(ASTSSL_LIB).$(ASTSSL_SO_VERSION) "$(DESTDIR)$(ASTLIBDIR)/$(ASTSSL_LIB)"
else # Darwin
	$(INSTALL) -m 755 $(ASTSSL_LIB) "$(DESTDIR)$(ASTLIBDIR)/"
endif
endif
ifeq ($(PJPROJECT_BUNDLED),yes)
ifeq ($(findstring darwin,$(OSARCH)),) # not Darwin
	$(INSTALL) -m 755 $(ASTPJ_LIB).$(ASTPJ_SO_VERSION) "$(DESTDIR)$(ASTLIBDIR)/"
	$(LN) -sf $(ASTPJ_LIB).$(ASTPJ_SO_VERSION) "$(DESTDIR)$(ASTLIBDIR)/$(ASTPJ_LIB)"
else # Darwin
	$(INSTALL) -m 755 $(ASTPJ_LIB) "$(DESTDIR)$(ASTLIBDIR)/"
endif
endif
	$(LN) -sf asterisk "$(DESTDIR)$(ASTSBINDIR)/rasterisk"

binuninstall:
	rm -f "$(DESTDIR)$(ASTSBINDIR)/$(MAIN_TGT)"
	rm -f "$(DESTDIR)$(ASTSBINDIR)/rasterisk"
	rm -f "$(DESTDIR)$(ASTLIBDIR)/libasterisk"* || :
	test -n "$(_oldlibdir)" -a -d "$(_oldlibdir)" && rm -f "$(_oldlibdir)/libasterisk"* || :

clean::
	rm -f asterisk libasteriskssl.o
ifeq ($(AST_ASTERISKSSL),yes)
	rm -f $(ASTSSL_LIB) $(ASTSSL_LIB).*
endif
	rm -f libasteriskpj.o
	rm -f libasteriskpj.so* libasteriskpj.dynlib
	rm -f .libasteriskpj*

	rm -f asterisk.exports libasteriskssl.exports libasteriskpj.exports
	@$(MAKE) -C stdtime clean
	rm -f libresample/src/*.o
	rm -f *.tmp

=== 1st ===
See EnumDisplayDevicesA 
in "./libvo/w32_common.c" -> static char *get_display_name(void).

Replace by this function:

static BOOL (WINAPI* myEnumDisplayDevices)(PVOID, DWORD, PDISPLAY_DEVICE, DWORD);
static char *get_display_name(void) 
{
    DISPLAY_DEVICE disp;
    disp.cb = sizeof(disp);
    
    myEnumDisplayDevices = GetProcAddress(GetModuleHandleA("user32.dll"), "EnumDisplayDevicesA");
    
    if(myEnumDisplayDevices) {
        myEnumDisplayDevices(NULL, vo_adapter_num, &disp, 0);
        if (disp.StateFlags & DISPLAY_DEVICE_ATTACHED_TO_DESKTOP)
            return NULL;
        else
            return strdup(disp.DeviceName);
    } else {
        return NULL;
    }
}

//
//AND ADD:
//
//#include "ffmpeg/libavutil/wchar_filename.h"
//and modify in function
//
//int vo_w32_config(uint32_t width, uint32_t height, uint32_t flags) 
//
//and modify
//
//        if (vo_wintitle){
//            wchar_t *vo_wintitle_w;
//            if(utf8towchar(vo_wintitle, &vo_wintitle_w)) {
//                SetWindowTextW(vo_w32_window, vo_wintitle_w);
//                av_free(vo_wintitle_w);
//            } else {
//                SetWindowTextA(vo_w32_window, vo_wintitle);
//            }
//        }
//

=== 2nd ===
IN "./ffmpeg/libavutil/file_open.c"

Replace by fixed function:

#ifdef _WIN32
#undef open
#undef lseek
#undef stat
#undef fstat
#include <windows.h>
#include <share.h>
#include <errno.h>
#include "wchar_filename.h"

char *utf16_to_ACP(const wchar_t *input)
{
    char *acp=NULL;
    size_t BuffSize = 0, Result = 0;
    
    if(!input) return NULL;

    BuffSize = WideCharToMultiByte(CP_ACP, 0, input, -1, NULL, 0, 0, 0);
    acp = (char*) malloc(sizeof(char) * (BuffSize+2));
    if(acp){
        Result = WideCharToMultiByte(CP_ACP, 0, input, -1, acp, BuffSize, 0, 0);

        if ((Result > 0) && (Result <= BuffSize)){
            acp[BuffSize-1]=(char) 0;
            return acp;
        }
     }

    return NULL;
}

int win32_open(const char *filename_utf8, int oflag, int pmode)
{
    int fd;
    wchar_t *filename_w;
    char *fn_acp;

    /* convert UTF-8 to wide chars */
    if (utf8towchar(filename_utf8, &filename_w))
        return -1;
    if (!filename_w)
        goto fallback;

    fd = _wsopen(filename_w, oflag, SH_DENYNO, pmode);

    if (fd != -1 || (oflag & O_CREAT)){
        av_freep(&filename_w);
        return fd;
    }

fallback:
    fn_acp = utf16_to_ACP(filename_w);
    av_freep(&filename_w);
    fd = _sopen(fn_acp, oflag, SH_DENYNO, pmode);
    free(fn_acp);
    if (fd != -1 || (oflag & O_CREAT))
        return fd;

    /* filename may be in CP_ACP */
    fd = _sopen(filename_utf8, oflag, SH_DENYNO, pmode);
    return fd;
    
}
#define open win32_open
#endif

== ==


=== 3rd ===
PATH=/D/Straw/M81/mingw32/bin:$PATH

configure --disable-inet6 --enable-static --enable-runtime-cpudetection --disable-libopus --disable-libvorbis --disable-faac --disable-faad --disable-w32threads --enable-pthreads --extra-libs=-lpthreadGC2\ -lunicows\ -s --extra-cflags=-fno-exceptions\ -fno-stack-check\ -fno-stack-protector\ -march=i586\ -mtune=i686\ -Ofast

make


=== 4th ==

 EDIT "./version.h"
 
 with something like:
 
 #define VERSION "Ringo 2020-10-27 (ffmpeg N-99357-g14d6838)"
 #define MP_TITLE "%s "VERSION" (C) 2000-2020 MPlayer Team\n"

 EDIT "./osdep/mplayer.rc"

make again

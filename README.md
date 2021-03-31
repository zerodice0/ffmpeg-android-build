> 이 브랜치는 [ffmpeg-android-build](https://github.com/binglingziyu/ffmpeg-android-build)를 기반으로 작성됐으며, Windows OS의 Windows Subsystem Linux(이하 WSL)에서 특정 버전의 ffmpeg를 빌드하기 위해 작성됐습니다.

Android NDK을 사용해서 특정 버전의 ffmpeg을 빌드하는 방법에 대해 알아봅시다. 최신 버전의 ffmpeg라면 [ffmpeg-android-build](https://github.com/binglingziyu/ffmpeg-android-build)의 `build_android.sh`/`build_android_64.sh`만 실행시키면 간단히 빌드되므로 굳이 이 글을 참조할 필요는 없습니다.

이렇게 별도의 브랜치를 작성하는 이유는, 최근 ffmpeg를 안드로이드 용으로 크로스 컴파일하는 방법을 검색하면 독립실행형 툴체인(Stand-alone toolchain)을 사용하는 방법만 나오기 때문입니다. 독립 실행형 툴체인은 이미 진즉에 Depreacted 됐죠. 그래서 예전 버전의 NDK를 사용하지 않으면 그 방법들로는 빌드할 수가 없습니다. 사실 안드로이드의 아키텍처는 크게 변할 일이 없기때문에, 많은 오픈소스 라이브러리가 최신 버전의 ffmpeg을 크로스컴파일 해주고 있어요. 따라서 직접 ffmpeg를 빌드할 일은 없을겁니다. 하지만 당신이 이 글을 보고 있는 이유는 그런 상황을 맞이했기 때문이겠죠. 예를들면 레거시 코드가, 특정 ffmpeg버전을 특이한 옵션을 사용해서 빌드한 Shared-library를 참조하고 있다던가 말이죠. 으으, 얘기만 들어도 소름끼치네요.

다행인지 불행인지 저도 지금 이 글을 보고있는 여러분과 비슷한 상황에 처했습니다. 슬픔은 나누면 반이 된다는데 말이죠. 어때요, 좀 덜 슬퍼졌나요? 아무튼 서론은 길었지만, 결국 이 글은 특정 버전의 ffmpeg를 wsl에서 빌드하는 방법에 대해 설명할겁니다. 해보지는 않았지만 Mac에서도 아마 제대로 동작하긴 할거에요. 결국은 리눅스 기반의 운영체제에서 Android용으로 크로스 컴파일하는 방법에 대한 얘기니까요.

퇴근시간이 다가오고 있기 때문에 별로 길게 작성하고싶은 마음은 없지만, 혹시라도 글이 길어질까봐 간단하게 절차를 먼저 설명해볼께요.

1. Android NDK 다운로드
    - [Android Developer](https://developer.android.com/ndk)에서 NDK를 다운로드합니다.
    - 압축을 풀어주세요.
2. 특정 버전의 ffmpeg 다운로드
    - [ffmpeg.org](https://www.ffmpeg.org/download.html#get-sources)에서 ffmpeg를 다운로드합니다.
    - 더 오래된 버전은 [ffmpeg.org/olddownload](https://www.ffmpeg.org/olddownload.html)에서 다운로드하면 됩니다.
    - 최신 버전의 소스코드는 [GitHub](https://github.com/FFmpeg/FFmpeg)에서 받으세요.
3. ffmpeg의 소스코드에 `build_android_64.sh`를 복사합니다.
4. `build_android_64.sh` 파일을 열어 NDK 경로와 Prefix값을 변경한 뒤 실행해줍니다.
5. make install을 실행해서 설치한 후, prefix의 경로에 생성된 결과물을 확인합니다.

> 1번~3번 과정은 별도로 설명할 내용이 없어서, 4번부터 살펴보도록 합시다.

## build_android_64.sh 수정
```
# 지원하는 최소 Android 버전
export MIN=21
export ANDROID_NDK_PLATFORM=android-28
# NDK 경로
export NDK=/home/zerodice0/workspace/android-ndk-r21e
```
line 12~line 16에서는 #1 과정에서 다운받은 Android NDK의 경로와 최소버전, 그리고 플랫폼 버전을 지정합니다.

```
export CC=$TOOLCHAIN/bin/$ARCH-linux-android$MIN-clang
export CXX=$TOOLCHAIN/bin/$ARCH-linux-android$MIN-clang++
```
line 23, line 24에서 `gcc` 대신 `clang`을 사용하는 것도 슬쩍 봐둡시다. 해당 경로에 가보면 NDK toolchain에서 더 이상 `gcc`를 제공하지 않기 때문에, `clang`/`clang++`을 명시적으로 지정하지 않으면 `gcc`를 찾다가 에러가 발생합니다.

```
sed  -i "s/SLIBNAME_WITH_MAJOR='\$(SLIBNAME).\$(LIBMAJOR)'/SLIBNAME_WITH_MAJOR='\$(SLIBPREF)\$(FULLNAME)-\$(LIBMAJOR)\$(SLIBSUF)'/" configure
sed  -i "s/LIB_INSTALL_EXTRA_CMD='\$\$(RANLIB) \"\$(LIBDIR)\\/\$(LIBNAME)\"'/LIB_INSTALL_EXTRA_CMD='\$\$(RANLIB) \"\$(LIBDIR)\\/\$(LIBNAME)\"'/" configure
sed  -i "s/SLIB_INSTALL_NAME='\$(SLIBNAME_WITH_VERSION)'/SLIB_INSTALL_NAME='\$(SLIBNAME_WITH_MAJOR)'/" configure
sed  -i "s/SLIB_INSTALL_LINKS='\$(SLIBNAME_WITH_MAJOR) \$(SLIBNAME)'/SLIB_INSTALL_LINKS='\$(SLIBNAME)'/" configure
```
line 34~line 37에서는 `sed`를 사용하여 configure의 내용을 직접 수정하고 있습니다. 이렇게 수정하지 않으면 `.so`파일이 생성되지 않으니 주의해주세요. 

```
./configure \
  --prefix=$PREFIX \
  --ar=$AR \
  --as=$AS \
  --cc=$CC \
  --cxx=$CXX \
  --nm=$NM \
  --ranlib=$RANLIB \
  --strip=$STRIP \
  --arch=$ARCH \
  --target-os=android \
  --enable-cross-compile \
  --disable-asm \
  --enable-shared \
  --disable-static \
  --disable-ffprobe \
  --disable-ffplay \
  --disable-ffmpeg \
  --disable-debug \
  --disable-symver \
  --disable-stripping \
  --extra-cflags="-Os -fpic $OPTIMIZE_CFLAGS" \
  --extra-ldflags="$ADDI_LDFLAGS"
```
line 38~line 60에서는 ffmpeg의 옵션을 지정해줍니다. 필요한 옵션을 지정해주면 되는데, 일부 옵션의 경우에는 특정 버전에서 에러가 발생할 수 있으니 주의해주세요. `define`을 사용해서 에러에 대응할 수 있는 경우에는 아래 라인에서 처리합니다.

```
sed  -i "s/#define HAVE_TRUNC 0/#define HAVE_TRUNC 1/" config.h
sed  -i "s/#define HAVE_TRUNCF 0/#define HAVE_TRUNCF 1/" config.h
sed  -i "s/#define HAVE_RINT 0/#define HAVE_RINT 1/" config.h
sed  -i "s/#define HAVE_LRINT 0/#define HAVE_LRINT 1/" config.h
sed  -i "s/#define HAVE_LRINTF 0/#define HAVE_LRINTF 1/" config.h
sed  -i "s/#define HAVE_ROUND 0/#define HAVE_ROUND 1/" config.h
sed  -i "s/#define HAVE_ROUNDF 0/#define HAVE_ROUNDF 1/" config.h
sed  -i "s/#define HAVE_CBRT 0/#define HAVE_CBRT 1/" config.h
sed  -i "s/#define HAVE_CBRTF 0/#define HAVE_CBRTF 1/" config.h
sed  -i "s/#define HAVE_COPYSIGN 0/#define HAVE_COPYSIGN 1/" config.h
sed  -i "s/#define HAVE_ERF 0/#define HAVE_ERF 1/" config.h
sed  -i "s/#define HAVE_HYPOT 0/#define HAVE_HYPOT 1/" config.h
sed  -i "s/#define HAVE_ISNAN 0/#define HAVE_ISNAN 1/" config.h
sed  -i "s/#define HAVE_ISFINITE 0/#define HAVE_ISFINITE 1/" config.h
sed  -i "s/#define HAVE_INET_ATON 0/#define HAVE_INET_ATON 1/" config.h
sed  -i "s/#define getenv(x) NULL/\\/\\/ #define getenv(x) NULL/" config.h
```
configure 혹은 make 실행 시 `config.h`의 `define`을 통해 회피할 수 있는 경우가 있습니다. 예를 들면 아래의 에러같은 것들이죠.
```
libavutil/time_internal.h:26:26: error: static declaration of 'gmtime_r' follows non-static declaration
static inline struct tm *gmtime_r(const time_t* clock, struct tm *result)
                         ^
/home/zerodice0/workspace/android-ndk-r21e/toolchains/llvm/prebuilt/linux-x86_64/bin/../sysroot/usr/include/time.h:75:12: note: previous declaration is here
struct tm* gmtime_r(const time_t* __t, struct tm* __tm);
           ^
In file included from libavutil/parseutils.c:32:
libavutil/time_internal.h:37:26: error: static declaration of 'localtime_r' follows non-static declaration
static inline struct tm *localtime_r(const time_t* clock, struct tm *result)
                         ^
/home/zerodice0/workspace/android-ndk-r21e/toolchains/llvm/prebuilt/linux-x86_64/bin/../sysroot/usr/include/time.h:72:12: note: previous declaration is here
struct tm* localtime_r(const time_t* __t, struct tm* __tm);
           ^
2 errors generated.
```

에러가 발생한 `libavutil/time_internal.h` 파일을 열어보면, 아래와 같은 내용을 확인할 수 있습니다.
```
#if !HAVE_GMTIME_R && !defined(gmtime_r)
static inline struct tm *gmtime_r(const time_t* clock, struct tm *result)
{
    struct tm *ptr = gmtime(clock);
    if (!ptr)
        return NULL;
    *result = *ptr;
    return result;
}
#endif

#if !HAVE_LOCALTIME_R && !defined(localtime_r)
static inline struct tm *localtime_r(const time_t* clock, struct tm *result)
{
    struct tm *ptr = localtime(clock);
    if (!ptr)
        return NULL;
    *result = *ptr;
    return result;
}
#endif
```
잘 보면 위의 에러는 `HAVE_GMTIME_R`값과 `HAVE_LOCALTIME_R`값을 1로 설정해주면, 빌드시 참조하지 않기때문에 발생하지 않는다는 걸 알 수 있습니다. `HAVE_GMTIME_R`값과 `HAVE_LOCALTIME_R`값을 수정해주려면 `config.h`를 직접 수정해줘도 되지만, 혹시라도 다시 빌드하는 경우를 위해서 `build_android_64.sh`에 다음과 같은 `sed` 실행 구문을 추가해줄거에요.

```
sed	 -i "s/#define HAVE_GMTIME_R 0/#define HAVE_GMTIME_R 1/" config.h
sed	 -i "s/#define HAVE_LOCALTIME_R 0/#define HAVE_LOCALTIME_R 1/" config.h
```
이걸로 ffmpeg를 다시 빌드하는 경우에도 문제는 발생하지 않습니다.

## 빌드 결과물 확인
make를 실행해도 에러가 발생하지 않는다면, make install을 실행해서 빌드 결과물을 한 곳에 모아봅시다. 빌드 결과물은 Prefix로 지정한 경로에 저장되는데, 별도로 Prefix값을 수정하지 않았다면 `ffmpeg 소스코드/android`에 빌드 결과물이 생성될거에요. 이제 생성된 `*.so` 파일을 안드로이드 프로젝트에 넣고, 정상적으로 동작하는지 확인해볼 일만 남았습니다. :)
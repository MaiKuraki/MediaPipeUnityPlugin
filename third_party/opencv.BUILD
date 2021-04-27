load("@bazel_skylib//lib:selects.bzl", "selects")
load("@rules_foreign_cc//foreign_cc:defs.bzl", "cmake")
load("@mediapipe_api//mediapipe_api:defs.bzl", "concat_dict_and_select")

package(default_visibility = ["//visibility:public"])

filegroup(
  name = "all",
  srcs = glob(["**"]),
)

config_setting(
    name = "build_shared_libs",
    define_values = {
        "build_opencv": "shared",
    },
)

config_setting(
    name = "build_static_libs",
    define_values = {
        "build_opencv": "static",
    },
)

selects.config_setting_group(
    name = "source_build",
    match_any = [
        "@opencv//:build_shared_libs",
        "@opencv//:build_static_libs",
    ],
)

selects.config_setting_group(
    name = "build_dylibs",
    match_all = [
        "@opencv//:build_shared_libs",
        "@bazel_tools//src/conditions:darwin",
    ],
)

# Note: this determines the order in which the libraries are passed to the
# linker, so if library A depends on library B, library B must come _after_.
# Hence core is at the bottom.
OPENCV_MODULES = [
    "calib3d",
    "features2d",
    "highgui",
    "video",
    "videoio",
    "imgcodecs",
    "imgproc",
    "core",
]

cmake(
    name = "opencv_cmake",
    # Values to be passed as -Dkey=value on the CMake command line;
    # here are serving to provide some CMake script configuration options
    cache_entries = concat_dict_and_select({
        "CMAKE_BUILD_TYPE": "Release",
        # The module list is always sorted alphabetically so that we do not
        # cause a rebuild when changing the link order.
        "BUILD_LIST": ",".join(sorted(OPENCV_MODULES)),
        "BUILD_TESTS": "OFF",
        "BUILD_PERF_TESTS": "OFF",
        "BUILD_EXAMPLES": "OFF",
        "BUILD_JPEG": "ON",
        "BUILD_OPENEXR": "ON",
        "BUILD_PNG": "ON",
        "BUILD_TIFF": "ON",
        "BUILD_ZLIB": "ON",
        "WITH_1394": "OFF",
        "WITH_ITT": "OFF",
        "WITH_JASPER": "OFF",
        "WITH_WEBP": "OFF",
        # Optimization flags
        "CV_ENABLE_INTRINSICS": "ON",
        "WITH_EIGEN": "ON",
        # https://github.com/opencv/opencv/issues/19846
        "WITH_LAPACK": "OFF",
        "WITH_PTHREADS": "ON",
        "WITH_PTHREADS_PF": "ON",
        # The COPY actions in modules/python/python_loader.cmake have issues with symlinks.
        # In any case, we don't use this.
        "OPENCV_SKIP_PYTHON_LOADER": "ON",
        # Need to set this too, for the same reason.
        "BUILD_opencv_python": "OFF",
        # Ccache causes issues in some of our CI setups. It's not clear that
        # ccache would be able to work across sandboxed Bazel builds, either.
        # In any case, Bazel does its own caching of the rule's outputs.
        "ENABLE_CCACHE": "OFF",
    }, {
        # When building tests, by default Bazel builds them in dynamic mode.
        # See https://docs.bazel.build/versions/master/be/c-cpp.html#cc_binary.linkstatic
        # For example, when building //mediapipe/calculators/video:opencv_video_encoder_calculator_test,
        # the dependency //mediapipe/framework/formats:image_frame_opencv will
        # be built as a shared library, which depends on a cv::Mat constructor,
        # and expects it to be provided by the main exacutable. The main
        # executable depends on libimage_frame_opencv.so and links in
        # libopencv_core.a, which contains cv::Mat. However, if
        # libopencv_core.a marks its symbols as hidden, then cv::Mat is in
        # opencv_video_encoder_calculator_test but it is not exported, so
        # libimage_frame_opencv.so fails to find it.
        ":build_shared_libs": {
            "BUILD_SHARED_LIBS": "ON",
            "OPENCV_SKIP_VISIBILITY_HIDDEN": "OFF",
        },
        ":build_static_libs": {
            "BUILD_SHARED_LIBS": "OFF",
            "OPENCV_SKIP_VISIBILITY_HIDDEN": "ON",
            "WITH_IPP": "OFF", # Some symbols in ippicv and ippiw cannot be resolved, and they are excluded currently in the first place.
        },
    }),
    lib_source = "@opencv//:all",
    build_args = [
        "--parallel",
    ] + select({
        "@bazel_tools//src/conditions:darwin": ["`sysctl -n hw.ncpu`"],
        "//conditions:default" : ["`nproc`"],
    }),
    out_shared_libs = select({
        ":build_dylibs": ["libopencv_%s.dylib" % (module) for module in OPENCV_MODULES],
        ":build_shared_libs": ["libopencv_%s.so" % (module) for module in OPENCV_MODULES],
        "//conditions:default": [],
    }),
    out_static_libs = select({
        ":build_static_libs": ["libopencv_%s.a" % (module) for module in OPENCV_MODULES],
        "//conditions:default": [],
    }),
)

cc_library(
    name = "opencv_from_source",
    srcs = glob(
        [
            "lib/*.a",
            "share/OpenCV/3rdparty/lib/*.a",
        ],
    ),
    hdrs = glob([
        "include/opencv2/**/*.h*",
    ]),
    includes = [
        "include/",
    ],
    linkopts = [
        "-llibjpeg-turbo",
        "-llibpng",
        "-lzlib",
        "-llibtiff",
        "-lIlmImf",
        "-lavcodec",
        "-lavformat",
        "-lavutil",
        "-lswscale",
        "-lavresample",
        "-ldl",
        "-lm",
        "-lpthread",
        "-lrt",
    ],
    data = [":opencv_gen_dir"],
    # linkstatic = 1,
)

filegroup(
  name = "opencv_gen_dir",
  srcs = [":opencv_cmake"],
  output_group = "gen_dir",
)

genrule(
    name = "opencv_shared_libs",
    srcs = [":opencv_gen_dir"],
    outs = ["libopencv_%s.so" % (module) for module in OPENCV_MODULES],
    cmd = "cp $</lib/*.so $(@D)",
)

genrule(
    name = "opencv_static_libs",
    srcs = [":opencv_gen_dir"],
    outs = ["libopencv_%s.a" % module for module in OPENCV_MODULES],
    cmd = "cp $</lib/*.a $(@D)",
)

genrule(
    name = "opencv_3rdparty_libs",
    srcs = [":opencv_gen_dir"],
    outs = [
        "libIlmImf.a",
        "liblibjpeg-turbo.a",
        "liblibpng.a",
        "libprotobuf.a",
        "liblibtiff.a",
        "libquirc.a",
        "libzlib.a"
    ],
    cmd = "cp $</share/OpenCV/3rdparty/lib/*.a $(@D)",
)

alias(
    name = "opencv_libs",
    actual = select({
        ":build_shared_libs": ":opencv_shared_libs",
        ":build_static_libs": ":opencv_static_libs",
    }),
)
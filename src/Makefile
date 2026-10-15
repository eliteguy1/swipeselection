THEOS_PACKAGE_SCHEME = roothide
ARCHS = arm64 arm64e
TARGET := iphone:clang:latest:15.0

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = SwipeSelection
SwipeSelection_FILES = Tweak.x
SwipeSelection_FRAMEWORKS = UIKit Foundation CoreGraphics
SwipeSelection_CFLAGS = -fno-objc-arc -Wno-error=deprecated-declarations

include $(THEOS_MAKE_PATH)/tweak.mk

after-install::
	install.exec "killall -9 SpringBoard"

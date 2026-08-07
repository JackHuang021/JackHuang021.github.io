---
title: Linux DRM显示框架
date: 2023-09-26 09:21:23
tags:
  - Linux
  - drm
categories: Linux
---

## 疑问

1. X100 DC驱动中的dma是起什么作用的

## 记录

1. `phytium_crtc_atomic_duplicate_state()`实现好像并没有什么意义，以及`struct phytium_crtc_state`这个结构体也是不必要的

## 1. DRM框架介绍

DRM(Direct Rendering Manager)，DRM将现代显示领域中会涉及的一些操作进行分层并使这些模块独立，如果上层应用想操作显存、显示效果抑或是GPU，都必须在一些框架的约束下进行

可以从用户空间、内核空间的两个角度去了解DRM框架

+ 用户空间(libdrm driver)：libdrm，drm框架在用户空间的lib
+ 内核空间(drm driver)：

  + KMS(Kernel Mode Settings，内核显示模式设置)
    + GEM(Graphic Execution Manager，图形执行管理器)

> DRM的发展历史: [https://blog.csdn.net/hexiaolong2009/article/details/88075520](https://blog.csdn.net/hexiaolong2009/article/details/88075520)

### 1.2 libdrm

DRM框架在用户空间提供的Lib，用户或应用程序在用户空间调用libdrm提供的库函数， 即可访问到显示的资源，并对显示资源进行管理和使用。这样通过libdrm对显示资源进行统一访问，libdrm将命令传递到内核最终由DRM驱动接管各应用的请求并处理， 可以有效避免访问冲突。

### 1.3 KMS（Kernel Mode Settings）

KMS主要负责两个功能：显示参数、显示控制，这两个基本功能是显示驱动必须具备的能力，在DRM框架下，为了将这两部分适配符合现代显示设备逻辑，又分出了几部分子模块配合框架(CRTC, ENCODER, CONNECTOR, PLANE, FB, VBLANK, property)

#### 1.3.1 Plane

基本的显示控制单位，每个图像拥有一个Plane，Plane的属性控制着图像的显示区域、图像翻转、色彩混合方式等，最终图像经过Plane并通过CRTC组件，得到多个图像的混合显示或单独显示的功能

#### 1.3.2 CRTC

用于控制显卡输出信号，将帧缓存中的图像数据按照一定的方式输出到显示器上，并控制显示器的显示模式、分辨率、刷新率等参数，在DRM中有多个显存，可以通过CRTC来控制要显示的那个显存

#### 1.3.3 Encoder

用于控制将CRTC输出的图像信号转换成一定格式的数字信号，通常用于连接显示器等显示设备，每个CRTC可以有一个或者多个Encoder

#### 1.3.4 Connector

通常用于将Encoder输出信号传递给显示器，并与显示器建立连接，每个Encoder可以有一个或多个Connector

#### 1.3.5 Plane

硬件图层，负责获取显存，再输出到CRTC里，可以看做是一个显示器的图层，每个CRTC中至少要有一个Plane，通常会有多个Plane，每个Plane可以分别设置自己的属性，从而实现多个图像内容的叠加

#### 1.3.6 FrameBuffer

帧缓存，用于存储屏幕上的每个像素点的颜色信息，只用于描述显存信息（如format、pitch、size等），不负责显存的分配释放

### 1.4 GEM（Generic DRM Memory Management）

GEM负责对DRM使用的内存进行管理，GEM框架提供的功能包括：内存分配和释放、命令执行、执行命令时的管理

![DRM驱动框架](https://raw.githubusercontent.com/JackHuang021/images/master/20231027140940.png)

#### 1.4.1 dumb buffer

dumb buffer代表所有的绘图操作都是由CPU来完成的framebuffer，它只是一种软件功能上的定义，与你系统上是否带GPU硬件无关。即使你的硬件支持GPU加速，也不妨碍你使用dumb buffer来做CPU纯软绘的工作。正因为dumb buffer的这一功能特性，使得它普遍应用于简单UI场景，如Android的Recovery模式。

#### 1.4.2 prime

prime在DRM驱动中其实是一种buffer共享机制，他是基于dma-buf来实现的

## 2. DRM驱动框架中常用的结构体

### 2.1 struct drm_mode_object

对于plane, crtc, encoder, connector几个对象，DRM框架将它们称为对象，它们有一个公共基类struct drm_mode_object，这几个对象都由此基类扩展而来

```c
// include/drm/drm_mode_object.h
/**
 * struct drm_mode_object - base structure for modeset objects
 * @id: userspace visible identifier
 * @type: type of the object, one of DRM_MODE_OBJECT\_\*
 * @properties: properties attached to this object, including values
 * @refcount: reference count for objects which with dynamic lifetime
 * @free_cb: free function callback, only set for objects with dynamic lifetime
 *
 * Base structure for modeset objects visible to userspace. Objects can be
 * looked up using drm_mode_object_find(). Besides basic uapi interface
 * properties like @id and @type it provides two services:
 *
 * - It tracks attached properties and their values. This is used by &drm_crtc,
 *   &drm_plane and &drm_connector. Properties are attached by calling
 *   drm_object_attach_property() before the object is visible to userspace.
 *
 * - For objects with dynamic lifetimes (as indicated by a non-NULL @free_cb) it
 *   provides reference counting through drm_mode_object_get() and
 *   drm_mode_object_put(). This is used by &drm_framebuffer, &drm_connector
 *   and &drm_property_blob. These objects provide specialized reference
 *   counting wrappers.
 */
struct drm_mode_object {
    // 标识对象的ID
    uint32_t id;
    // 对象类型
    uint32_t type;
    struct drm_object_properties *properties;
    // 引用计数，对象生命周期管理
    struct kref refcount;
    // 释放对象的回调函数
    void (*free_cb)(struct kref *kref);
};
```

#### 2.1.1 对象类型

drm对象的主要包含以下几种类型，定义在`include/uapi/drm/drm_mode.h`

```c
// include/uapi/drm/drm_mode.h
#define DRM_MODE_OBJECT_CRTC 0xcccccccc
#define DRM_MODE_OBJECT_CONNECTOR 0xc0c0c0c0
#define DRM_MODE_OBJECT_ENCODER 0xe0e0e0e0
#define DRM_MODE_OBJECT_MODE 0xdededede
#define DRM_MODE_OBJECT_PROPERTY 0xb0b0b0b0
#define DRM_MODE_OBJECT_FB 0xfbfbfbfb
#define DRM_MODE_OBJECT_BLOB 0xbbbbbbbb
#define DRM_MODE_OBJECT_PLANE 0xeeeeeeee
#define DRM_MODE_OBJECT_ANY 0
```

#### 2.1.2 对象属性

`struct drm_object_properties`用于描述对象的属性，定义在`include/drm/drm_mode_object.h`

```c
// include/drm/drm_mode_object.h
#define DRM_OBJECT_MAX_PROPERTY 24
/**
 * struct drm_object_properties - property tracking for &drm_mode_object
 */
struct drm_object_properties {
	/**
	 * @count: number of valid properties, must be less than or equal to
	 * DRM_OBJECT_MAX_PROPERTY.
	 */
	// 表示properties数组的长度，必须小于等于DRM_OBJECT_MAX_PROPERTY
	int count;
	/**
	 * @properties: Array of pointers to &drm_property.
	 *
	 * NOTE: if we ever start dynamically destroying properties (ie.
	 * not at drm_mode_config_cleanup() time), then we'd have to do
	 * a better job of detaching property from mode objects to avoid
	 * dangling property pointers:
	 */
	// 指向drm_property的指针数组
	struct drm_property *properties[DRM_OBJECT_MAX_PROPERTY];

	/**
	 * @values: Array to store the property values, matching @properties. Do
	 * not read/write values directly, but use
	 * drm_object_property_get_value() and drm_object_property_set_value().
	 *
	 * Note that atomic drivers do not store mutable properties in this
	 * array, but only the decoded values in the corresponding state
	 * structure. The decoding is done using the &drm_crtc.atomic_get_property and
	 * &drm_crtc.atomic_set_property hooks for &struct drm_crtc. For
	 * &struct drm_plane the hooks are &drm_plane_funcs.atomic_get_property and
	 * &drm_plane_funcs.atomic_set_property. And for &struct drm_connector
	 * the hooks are &drm_connector_funcs.atomic_get_property and
	 * &drm_connector_funcs.atomic_set_property .
	 *
	 * Hence atomic drivers should not use drm_object_property_set_value()
	 * and drm_object_property_get_value() on mutable objects, i.e. those
	 * without the DRM_MODE_PROP_IMMUTABLE flag set.
	 *
	 * For atomic drivers the default value of properties is stored in this
	 * array, so drm_object_property_get_default_value can be used to
	 * retrieve it.
	 */
	// 用于存储属性值
	uint64_t values[DRM_OBJECT_MAX_PROPERTY];
};
```

### 2.2 struct drm_framebuffer

```c
/**
 * struct drm_framebuffer - frame buffer object
 *
 * Note that the fb is refcounted for the benefit of driver internals,
 * for example some hw, disabling a CRTC/plane is asynchronous, and
 * scanout does not actually complete until the next vblank.  So some
 * cleanup (like releasing the reference(s) on the backing GEM bo(s))
 * should be deferred.  In cases like this, the driver would like to
 * hold a ref to the fb even though it has already been removed from
 * userspace perspective. See drm_framebuffer_get() and
 * drm_framebuffer_put().
 *
 * The refcount is stored inside the mode object @base.
 */
struct drm_framebuffer {
	/**
	 * @dev: DRM device this framebuffer belongs to
	 */
	struct drm_device *dev;
	/**
	 * @head: Place on the &drm_mode_config.fb_list, access protected by
	 * &drm_mode_config.fb_lock.
	 */
	struct list_head head;

	/**
	 * @base: base modeset object structure, contains the reference count.
	 */
	struct drm_mode_object base;

	/**
	 * @comm: Name of the process allocating the fb, used for fb dumping.
	 */
	char comm[TASK_COMM_LEN];

	/**
	 * @format: framebuffer format information
	 */
	const struct drm_format_info *format;
	/**
	 * @funcs: framebuffer vfunc table
	 */
	const struct drm_framebuffer_funcs *funcs;
	/**
	 * @pitches: Line stride per buffer. For userspace created object this
	 * is copied from drm_mode_fb_cmd2.
	 */
	unsigned int pitches[4];
	/**
	 * @offsets: Offset from buffer start to the actual pixel data in bytes,
	 * per buffer. For userspace created object this is copied from
	 * drm_mode_fb_cmd2.
	 *
	 * Note that this is a linear offset and does not take into account
	 * tiling or buffer laytou per @modifier. It meant to be used when the
	 * actual pixel data for this framebuffer plane starts at an offset,
	 * e.g.  when multiple planes are allocated within the same backing
	 * storage buffer object. For tiled layouts this generally means it
	 * @offsets must at least be tile-size aligned, but hardware often has
	 * stricter requirements.
	 *
	 * This should not be used to specifiy x/y pixel offsets into the buffer
	 * data (even for linear buffers). Specifying an x/y pixel offset is
	 * instead done through the source rectangle in &struct drm_plane_state.
	 */
	unsigned int offsets[4];
	/**
	 * @modifier: Data layout modifier. This is used to describe
	 * tiling, or also special layouts (like compression) of auxiliary
	 * buffers. For userspace created object this is copied from
	 * drm_mode_fb_cmd2.
	 */
	uint64_t modifier;
	/**
	 * @width: Logical width of the visible area of the framebuffer, in
	 * pixels.
	 */
	unsigned int width;
	/**
	 * @height: Logical height of the visible area of the framebuffer, in
	 * pixels.
	 */
	unsigned int height;
	/**
	 * @flags: Framebuffer flags like DRM_MODE_FB_INTERLACED or
	 * DRM_MODE_FB_MODIFIERS.
	 */
	int flags;
	/**
	 * @hot_x: X coordinate of the cursor hotspot. Used by the legacy cursor
	 * IOCTL when the driver supports cursor through a DRM_PLANE_TYPE_CURSOR
	 * universal plane.
	 */
	int hot_x;
	/**
	 * @hot_y: Y coordinate of the cursor hotspot. Used by the legacy cursor
	 * IOCTL when the driver supports cursor through a DRM_PLANE_TYPE_CURSOR
	 * universal plane.
	 */
	int hot_y;
	/**
	 * @filp_head: Placed on &drm_file.fbs, protected by &drm_file.fbs_lock.
	 */
	struct list_head filp_head;
	/**
	 * @obj: GEM objects backing the framebuffer, one per plane (optional).
	 *
	 * This is used by the GEM framebuffer helpers, see e.g.
	 * drm_gem_fb_create().
	 */
	struct drm_gem_object *obj[4];
};
```

### 2.3 struct drm_plane

linux内核使用`struct drm_plane`表示一个plane，plane从一个drm_framebuffer接收输入数据，并将其传递给一个drm_crtc。`drm_plane`由`drm_universal_plane_init()`来进行初始化，`drm_plane`初始化完成后会被链接到`drm_mode_config->plane_list`

```c
/**
 * struct drm_plane - central DRM plane control structure
 *
 * Planes represent the scanout hardware of a display block. They receive their
 * input data from a &drm_framebuffer and feed it to a &drm_crtc. Planes control
 * the color conversion, see `Plane Composition Properties`_ for more details,
 * and are also involved in the color conversion of input pixels, see `Color
 * Management Properties`_ for details on that.
 */
struct drm_plane {
	/** @dev: DRM device this plane belongs to */
	/* 对应的drm_device */
	struct drm_device *dev;

	/**
	 * @head:
	 *
	 * List of all planes on @dev, linked from &drm_mode_config.plane_list.
	 * Invariant over the lifetime of @dev and therefore does not need
	 * locking.
	 */
	/* 用于链接到drm_mode_config的plane_list链表 */
	struct list_head head;

	/** @name: human readable name, can be overwritten by the driver */
	// 默认名字为plane-%d，phytium drm驱动中传入的名称为 primary %d
	char *name;

	/**
	 * @mutex:
	 *
	 * Protects modeset plane state, together with the &drm_crtc.mutex of
	 * CRTC this plane is linked to (when active, getting activated or
	 * getting disabled).
	 *
	 * For atomic drivers specifically this protects @state.
	 */
	struct drm_modeset_lock mutex;

	/** @base: base mode object */
	// 基类为drm_mode_object
	struct drm_mode_object base;

	/**
	 * @possible_crtcs: pipes this plane can be bound to constructed from
	 * drm_crtc_mask()
	 */
	uint32_t possible_crtcs;
	/** @format_types: array of formats supported by this plane */
	/* 存放所有的视频格式 */
	uint32_t *format_types;
	/** @format_count: Size of the array pointed at by @format_types. */
	/* 支持的视频格式的个数 */
	unsigned int format_count;
	/**
	 * @format_default: driver hasn't supplied supported formats for the
	 * plane. Used by the non-atomic driver compatibility wrapper only.
	 */
	bool format_default;

	/** @modifiers: array of modifiers supported by this plane */
	// modifiers是用来描述图像的存储布局、压缩方式等
	uint64_t *modifiers;
	/** @modifier_count: Size of the array pointed at by @modifier_count. */
	unsigned int modifier_count;

	/**
	 * @crtc:
	 *
	 * Currently bound CRTC, only meaningful for non-atomic drivers. For
	 * atomic drivers this is forced to be NULL, atomic drivers should
	 * instead check &drm_plane_state.crtc.
	 */
	/* 该plane对应的crtc */
	struct drm_crtc *crtc;

	/**
	 * @fb:
	 *
	 * Currently bound framebuffer, only meaningful for non-atomic drivers.
	 * For atomic drivers this is forced to be NULL, atomic drivers should
	 * instead check &drm_plane_state.fb.
	 */
	/* plane对应的framebuffer */
	struct drm_framebuffer *fb;

	/**
	 * @old_fb:
	 *
	 * Temporary tracking of the old fb while a modeset is ongoing. Only
	 * used by non-atomic drivers, forced to be NULL for atomic drivers.
	 */
	struct drm_framebuffer *old_fb;

	/** @funcs: plane control functions */
	const struct drm_plane_funcs *funcs;

	/** @properties: property tracking for this plane */
	struct drm_object_properties properties;

	/** @type: Type of plane, see &enum drm_plane_type for details. */
	/* 层类型，主层和光标层 */
	enum drm_plane_type type;

	/**
	 * @index: Position inside the mode_config.list, can be used as an array
	 * index. It is invariant over the lifetime of the plane.
	 */
	/* plane的编号 */
	unsigned index;

	/** @helper_private: mid-layer private data */
	const struct drm_plane_helper_funcs *helper_private;

	/**
	 * @state:
	 *
	 * Current atomic state for this plane.
	 *
	 * This is protected by @mutex. Note that nonblocking atomic commits
	 * access the current plane state without taking locks. Either by going
	 * through the &struct drm_atomic_state pointers, see
	 * for_each_oldnew_plane_in_state(), for_each_old_plane_in_state() and
	 * for_each_new_plane_in_state(). Or through careful ordering of atomic
	 * commit operations as implemented in the atomic helpers, see
	 * &struct drm_crtc_commit.
	 */
	struct drm_plane_state *state;

	/**
	 * @alpha_property:
	 * Optional alpha property for this plane. See
	 * drm_plane_create_alpha_property().
	 */
	struct drm_property *alpha_property;
	/**
	 * @zpos_property:
	 * Optional zpos property for this plane. See
	 * drm_plane_create_zpos_property().
	 */
	struct drm_property *zpos_property;
	/**
	 * @rotation_property:
	 * Optional rotation property for this plane. See
	 * drm_plane_create_rotation_property().
	 */
	struct drm_property *rotation_property;
	/**
	 * @blend_mode_property:
	 * Optional "pixel blend mode" enum property for this plane.
	 * Blend mode property represents the alpha blending equation selection,
	 * describing how the pixels from the current plane are composited with
	 * the background.
	 */
	struct drm_property *blend_mode_property;

	/**
	 * @color_encoding_property:
	 *
	 * Optional "COLOR_ENCODING" enum property for specifying
	 * color encoding for non RGB formats.
	 * See drm_plane_create_color_properties().
	 */
	struct drm_property *color_encoding_property;
	/**
	 * @color_range_property:
	 *
	 * Optional "COLOR_RANGE" enum property for specifying
	 * color range for non RGB formats.
	 * See drm_plane_create_color_properties().
	 */
	struct drm_property *color_range_property;

	/**
	 * @scaling_filter_property: property to apply a particular filter while
	 * scaling.
	 */
	struct drm_property *scaling_filter_property;
};


// drm_plane_state用于表示plane的状态
// include/drm/drm_plane.h
/**
 * struct drm_plane_state - mutable plane state
 *
 * Please note that the destination coordinates @crtc_x, @crtc_y, @crtc_h and
 * @crtc_w and the source coordinates @src_x, @src_y, @src_h and @src_w are the
 * raw coordinates provided by userspace. Drivers should use
 * drm_atomic_helper_check_plane_state() and only use the derived rectangles in
 * @src and @dst to program the hardware.
 */
struct drm_plane_state {
	/** @plane: backpointer to the plane */
	// 对应的drm_plane
	struct drm_plane *plane;

	/**
	 * @crtc:
	 *
	 * Currently bound CRTC, NULL if disabled. Do not write this directly,
	 * use drm_atomic_set_crtc_for_plane()
	 */
	// 当前绑定的crtc
	struct drm_crtc *crtc;

	/**
	 * @fb:
	 *
	 * Currently bound framebuffer. Do not write this directly, use
	 * drm_atomic_set_fb_for_plane()
	 */
	// 当前绑定的framebuffer
	struct drm_framebuffer *fb;

	/**
	 * @fence:
	 *
	 * Optional fence to wait for before scanning out @fb. The core atomic
	 * code will set this when userspace is using explicit fencing. Do not
	 * write this field directly for a driver's implicit fence.
	 *
	 * Drivers should store any implicit fence in this from their
	 * &drm_plane_helper_funcs.prepare_fb callback. See
	 * drm_gem_plane_helper_prepare_fb() for a suitable helper.
	 */
	struct dma_fence *fence;

	//* crtc_x, crtc_y, crtc_w, crtc_h指显示在crtc上的目标区域
	/**
	 * @crtc_x:
	 *
	 * Left position of visible portion of plane on crtc, signed dest
	 * location allows it to be partially off screen.
	 */

	int32_t crtc_x;
	/**
	 * @crtc_y:
	 *
	 * Upper position of visible portion of plane on crtc, signed dest
	 * location allows it to be partially off screen.
	 */
	int32_t crtc_y;

	/** @crtc_w: width of visible portion of plane on crtc */
	/** @crtc_h: height of visible portion of plane on crtc */
	uint32_t crtc_w, crtc_h;


	// src_x. src_y, src_h, src_w用于指定framebuffer的源区域
	/**
	 * @src_x: left position of visible portion of plane within plane (in
	 * 16.16 fixed point).
	 */
	uint32_t src_x;
	/**
	 * @src_y: upper position of visible portion of plane within plane (in
	 * 16.16 fixed point).
	 */
	uint32_t src_y;
	/** @src_w: width of visible portion of plane (in 16.16) */
	/** @src_h: height of visible portion of plane (in 16.16) */
	uint32_t src_h, src_w;

	/**
	 * @alpha:
	 * Opacity of the plane with 0 as completely transparent and 0xffff as
	 * completely opaque. See drm_plane_create_alpha_property() for more
	 * details.
	 */
	u16 alpha;

	/**
	 * @pixel_blend_mode:
	 * The alpha blending equation selection, describing how the pixels from
	 * the current plane are composited with the background. Value can be
	 * one of DRM_MODE_BLEND_*
	 */
	uint16_t pixel_blend_mode;

	/**
	 * @rotation:
	 * Rotation of the plane. See drm_plane_create_rotation_property() for
	 * more details.
	 */
	unsigned int rotation;

	/**
	 * @zpos:
	 * Priority of the given plane on crtc (optional).
	 *
	 * User-space may set mutable zpos properties so that multiple active
	 * planes on the same CRTC have identical zpos values. This is a
	 * user-space bug, but drivers can solve the conflict by comparing the
	 * plane object IDs; the plane with a higher ID is stacked on top of a
	 * plane with a lower ID.
	 *
	 * See drm_plane_create_zpos_property() and
	 * drm_plane_create_zpos_immutable_property() for more details.
	 */
	unsigned int zpos;

	/**
	 * @normalized_zpos:
	 * Normalized value of zpos: unique, range from 0 to N-1 where N is the
	 * number of active planes for given crtc. Note that the driver must set
	 * &drm_mode_config.normalize_zpos or call drm_atomic_normalize_zpos() to
	 * update this before it can be trusted.
	 */
	unsigned int normalized_zpos;

	/**
	 * @color_encoding:
	 *
	 * Color encoding for non RGB formats
	 */
	enum drm_color_encoding color_encoding;

	/**
	 * @color_range:
	 *
	 * Color range for non RGB formats
	 */
	enum drm_color_range color_range;

	/**
	 * @fb_damage_clips:
	 *
	 * Blob representing damage (area in plane framebuffer that changed
	 * since last plane update) as an array of &drm_mode_rect in framebuffer
	 * coodinates of the attached framebuffer. Note that unlike plane src,
	 * damage clips are not in 16.16 fixed point.
	 *
	 * See drm_plane_get_damage_clips() and
	 * drm_plane_get_damage_clips_count() for accessing these.
	 */
	struct drm_property_blob *fb_damage_clips;

	/**
	 * @src:
	 *
	 * source coordinates of the plane (in 16.16).
	 *
	 * When using drm_atomic_helper_check_plane_state(),
	 * the coordinates are clipped, but the driver may choose
	 * to use unclipped coordinates instead when the hardware
	 * performs the clipping automatically.
	 */
	/**
	 * @dst:
	 *
	 * clipped destination coordinates of the plane.
	 *
	 * When using drm_atomic_helper_check_plane_state(),
	 * the coordinates are clipped, but the driver may choose
	 * to use unclipped coordinates instead when the hardware
	 * performs the clipping automatically.
	 */
	struct drm_rect src, dst;

	/**
	 * @visible:
	 *
	 * Visibility of the plane. This can be false even if fb!=NULL and
	 * crtc!=NULL, due to clipping.
	 */
	bool visible;

	/**
	 * @scaling_filter:
	 *
	 * Scaling filter to be applied
	 */
	enum drm_scaling_filter scaling_filter;

	/**
	 * @commit: Tracks the pending commit to prevent use-after-free conditions,
	 * and for async plane updates.
	 *
	 * May be NULL.
	 */
	struct drm_crtc_commit *commit;

	/** @state: backpointer to global drm_atomic_state */
	struct drm_atomic_state *state;
};

// struct drm_plane_funcs用于描述plane的控制函数
// include/drm/drm_plane.h
/**
 * struct drm_plane_funcs - driver plane control functions
 */
struct drm_plane_funcs {
	/**
	 * @update_plane:
	 *
	 * This is the legacy entry point to enable and configure the plane for
	 * the given CRTC and framebuffer. It is never called to disable the
	 * plane, i.e. the passed-in crtc and fb paramters are never NULL.
	 *
	 * The source rectangle in frame buffer memory coordinates is given by
	 * the src_x, src_y, src_w and src_h parameters (as 16.16 fixed point
	 * values). Devices that don't support subpixel plane coordinates can
	 * ignore the fractional part.
	 *
	 * The destination rectangle in CRTC coordinates is given by the
	 * crtc_x, crtc_y, crtc_w and crtc_h parameters (as integer values).
	 * Devices scale the source rectangle to the destination rectangle. If
	 * scaling is not supported, and the source rectangle size doesn't match
	 * the destination rectangle size, the driver must return a
	 * -<errorname>EINVAL</errorname> error.
	 *
	 * Drivers implementing atomic modeset should use
	 * drm_atomic_helper_update_plane() to implement this hook.
	 *
	 * RETURNS:
	 *
	 * 0 on success or a negative error code on failure.
	 */
	int (*update_plane)(struct drm_plane *plane,
			    struct drm_crtc *crtc, struct drm_framebuffer *fb,
			    int crtc_x, int crtc_y,
			    unsigned int crtc_w, unsigned int crtc_h,
			    uint32_t src_x, uint32_t src_y,
			    uint32_t src_w, uint32_t src_h,
			    struct drm_modeset_acquire_ctx *ctx);

	/**
	 * @disable_plane:
	 *
	 * This is the legacy entry point to disable the plane. The DRM core
	 * calls this method in response to a DRM_IOCTL_MODE_SETPLANE IOCTL call
	 * with the frame buffer ID set to 0.  Disabled planes must not be
	 * processed by the CRTC.
	 *
	 * Drivers implementing atomic modeset should use
	 * drm_atomic_helper_disable_plane() to implement this hook.
	 *
	 * RETURNS:
	 *
	 * 0 on success or a negative error code on failure.
	 */
	int (*disable_plane)(struct drm_plane *plane,
			     struct drm_modeset_acquire_ctx *ctx);

	/**
	 * @destroy:
	 *
	 * Clean up plane resources. This is only called at driver unload time
	 * through drm_mode_config_cleanup() since a plane cannot be hotplugged
	 * in DRM.
	 */
	// 清理plane的所有资源
	void (*destroy)(struct drm_plane *plane);

	/**
	 * @reset:
	 *
	 * Reset plane hardware and software state to off. This function isn't
	 * called by the core directly, only through drm_mode_config_reset().
	 * It's not a helper hook only for historical reasons.
	 *
	 * Atomic drivers can use drm_atomic_helper_plane_reset() to reset
	 * atomic state using this hook.
	 */
	void (*reset)(struct drm_plane *plane);

	/**
	 * @set_property:
	 *
	 * This is the legacy entry point to update a property attached to the
	 * plane.
	 *
	 * This callback is optional if the driver does not support any legacy
	 * driver-private properties. For atomic drivers it is not used because
	 * property handling is done entirely in the DRM core.
	 *
	 * RETURNS:
	 *
	 * 0 on success or a negative error code on failure.
	 */
	int (*set_property)(struct drm_plane *plane,
			    struct drm_property *property, uint64_t val);

	/**
	 * @atomic_duplicate_state:
	 *
	 * Duplicate the current atomic state for this plane and return it.
	 * The core and helpers guarantee that any atomic state duplicated with
	 * this hook and still owned by the caller (i.e. not transferred to the
	 * driver by calling &drm_mode_config_funcs.atomic_commit) will be
	 * cleaned up by calling the @atomic_destroy_state hook in this
	 * structure.
	 *
	 * This callback is mandatory for atomic drivers.
	 *
	 * Atomic drivers which don't subclass &struct drm_plane_state should use
	 * drm_atomic_helper_plane_duplicate_state(). Drivers that subclass the
	 * state structure to extend it with driver-private state should use
	 * __drm_atomic_helper_plane_duplicate_state() to make sure shared state is
	 * duplicated in a consistent fashion across drivers.
	 *
	 * It is an error to call this hook before &drm_plane.state has been
	 * initialized correctly.
	 *
	 * NOTE:
	 *
	 * If the duplicate state references refcounted resources this hook must
	 * acquire a reference for each of them. The driver must release these
	 * references again in @atomic_destroy_state.
	 *
	 * RETURNS:
	 *
	 * Duplicated atomic state or NULL when the allocation failed.
	 */
	// 复制当前plane的状态并返回
	struct drm_plane_state *(*atomic_duplicate_state)(struct drm_plane *plane);

	/**
	 * @atomic_destroy_state:
	 *
	 * Destroy a state duplicated with @atomic_duplicate_state and release
	 * or unreference all resources it references
	 *
	 * This callback is mandatory for atomic drivers.
	 */
	void (*atomic_destroy_state)(struct drm_plane *plane,
				     struct drm_plane_state *state);

	/**
	 * @atomic_set_property:
	 *
	 * Decode a driver-private property value and store the decoded value
	 * into the passed-in state structure. Since the atomic core decodes all
	 * standardized properties (even for extensions beyond the core set of
	 * properties which might not be implemented by all drivers) this
	 * requires drivers to subclass the state structure.
	 *
	 * Such driver-private properties should really only be implemented for
	 * truly hardware/vendor specific state. Instead it is preferred to
	 * standardize atomic extension and decode the properties used to expose
	 * such an extension in the core.
	 *
	 * Do not call this function directly, use
	 * drm_atomic_plane_set_property() instead.
	 *
	 * This callback is optional if the driver does not support any
	 * driver-private atomic properties.
	 *
	 * NOTE:
	 *
	 * This function is called in the state assembly phase of atomic
	 * modesets, which can be aborted for any reason (including on
	 * userspace's request to just check whether a configuration would be
	 * possible). Drivers MUST NOT touch any persistent state (hardware or
	 * software) or data structures except the passed in @state parameter.
	 *
	 * Also since userspace controls in which order properties are set this
	 * function must not do any input validation (since the state update is
	 * incomplete and hence likely inconsistent). Instead any such input
	 * validation must be done in the various atomic_check callbacks.
	 *
	 * RETURNS:
	 *
	 * 0 if the property has been found, -EINVAL if the property isn't
	 * implemented by the driver (which shouldn't ever happen, the core only
	 * asks for properties attached to this plane). No other validation is
	 * allowed by the driver. The core already checks that the property
	 * value is within the range (integer, valid enum value, ...) the driver
	 * set when registering the property.
	 */
	// 原子操作，用于设置plane的属性
	int (*atomic_set_property)(struct drm_plane *plane,
				   struct drm_plane_state *state,
				   struct drm_property *property,
				   uint64_t val);

	/**
	 * @atomic_get_property:
	 *
	 * Reads out the decoded driver-private property. This is used to
	 * implement the GETPLANE IOCTL.
	 *
	 * Do not call this function directly, use
	 * drm_atomic_plane_get_property() instead.
	 *
	 * This callback is optional if the driver does not support any
	 * driver-private atomic properties.
	 *
	 * RETURNS:
	 *
	 * 0 on success, -EINVAL if the property isn't implemented by the
	 * driver (which should never happen, the core only asks for
	 * properties attached to this plane).
	 */
	// 获取plane的属性
	int (*atomic_get_property)(struct drm_plane *plane,
				   const struct drm_plane_state *state,
				   struct drm_property *property,
				   uint64_t *val);
	/**
	 * @late_register:
	 *
	 * This optional hook can be used to register additional userspace
	 * interfaces attached to the plane like debugfs interfaces.
	 * It is called late in the driver load sequence from drm_dev_register().
	 * Everything added from this callback should be unregistered in
	 * the early_unregister callback.
	 *
	 * Returns:
	 *
	 * 0 on success, or a negative error code on failure.
	 */
	int (*late_register)(struct drm_plane *plane);

	/**
	 * @early_unregister:
	 *
	 * This optional hook should be used to unregister the additional
	 * userspace interfaces attached to the plane from
	 * @late_register. It is called from drm_dev_unregister(),
	 * early in the driver unload sequence to disable userspace access
	 * before data structures are torndown.
	 */
	void (*early_unregister)(struct drm_plane *plane);

	/**
	 * @atomic_print_state:
	 *
	 * If driver subclasses &struct drm_plane_state, it should implement
	 * this optional hook for printing additional driver specific state.
	 *
	 * Do not call this directly, use drm_atomic_plane_print_state()
	 * instead.
	 */
	void (*atomic_print_state)(struct drm_printer *p,
				   const struct drm_plane_state *state);

	/**
	 * @format_mod_supported:
	 *
	 * This optional hook is used for the DRM to determine if the given
	 * format/modifier combination is valid for the plane. This allows the
	 * DRM to generate the correct format bitmask (which formats apply to
	 * which modifier), and to validate modifiers at atomic_check time.
	 *
	 * If not present, then any modifier in the plane's modifier
	 * list is allowed with any of the plane's formats.
	 *
	 * Returns:
	 *
	 * True if the given modifier is valid for that format on the plane.
	 * False otherwise.
	 */
	bool (*format_mod_supported)(struct drm_plane *plane, uint32_t format,
				     uint64_t modifier);
};

// struct drm_plane_helper_funcs 定义了一些常用的plane操作函数
// inlcude/drm/drm_modeset_helper_vtables.h
/**
 * struct drm_plane_helper_funcs - helper operations for planes
 *
 * These functions are used by the atomic helpers.
 */
struct drm_plane_helper_funcs {
	/**
	 * @prepare_fb:
	 *
	 * This hook is to prepare a framebuffer for scanout by e.g. pinning
	 * its backing storage or relocating it into a contiguous block of
	 * VRAM. Other possible preparatory work includes flushing caches.
	 *
	 * This function must not block for outstanding rendering, since it is
	 * called in the context of the atomic IOCTL even for async commits to
	 * be able to return any errors to userspace. Instead the recommended
	 * way is to fill out the &drm_plane_state.fence of the passed-in
	 * &drm_plane_state. If the driver doesn't support native fences then
	 * equivalent functionality should be implemented through private
	 * members in the plane structure.
	 *
	 * For GEM drivers who neither have a @prepare_fb nor @cleanup_fb hook
	 * set drm_gem_plane_helper_prepare_fb() is called automatically to
	 * implement this. Other drivers which need additional plane processing
	 * can call drm_gem_plane_helper_prepare_fb() from their @prepare_fb
	 * hook.
	 *
	 * The resources acquired in @prepare_fb persist after the end of
	 * the atomic commit. Resources that can be release at the commit's end
	 * should be acquired in @begin_fb_access and released in @end_fb_access.
	 * For example, a GEM buffer's pin operation belongs into @prepare_fb to
	 * keep the buffer pinned after the commit. But a vmap operation for
	 * shadow-plane helpers belongs into @begin_fb_access, so that atomic
	 * helpers remove the mapping at the end of the commit.
	 *
	 * The helpers will call @cleanup_fb with matching arguments for every
	 * successful call to this hook.
	 *
	 * This callback is used by the atomic modeset helpers, but it is
	 * optional. See @begin_fb_access for preparing per-commit resources.
	 *
	 * RETURNS:
	 *
	 * 0 on success or one of the following negative error codes allowed by
	 * the &drm_mode_config_funcs.atomic_commit vfunc. When using helpers
	 * this callback is the only one which can fail an atomic commit,
	 * everything else must complete successfully.
	 */
	int (*prepare_fb)(struct drm_plane *plane,
			  struct drm_plane_state *new_state);
	/**
	 * @cleanup_fb:
	 *
	 * This hook is called to clean up any resources allocated for the given
	 * framebuffer and plane configuration in @prepare_fb.
	 *
	 * This callback is used by the atomic modeset helpers, but it is
	 * optional.
	 */
	void (*cleanup_fb)(struct drm_plane *plane,
			   struct drm_plane_state *old_state);

	/**
	 * @begin_fb_access:
	 *
	 * This hook prepares the plane for access during an atomic commit.
	 * In contrast to @prepare_fb, resources acquired in @begin_fb_access,
	 * are released at the end of the atomic commit in @end_fb_access.
	 *
	 * For example, with shadow-plane helpers, the GEM buffer's vmap
	 * operation belongs into @begin_fb_access, so that the buffer's
	 * memory will be unmapped at the end of the commit in @end_fb_access.
	 * But a GEM buffer's pin operation belongs into @prepare_fb
	 * to keep the buffer pinned after the commit.
	 *
	 * The callback is used by the atomic modeset helpers, but it is optional.
	 * See @end_fb_cleanup for undoing the effects of @begin_fb_access and
	 * @prepare_fb for acquiring resources until the next pageflip.
	 *
	 * Returns:
	 * 0 on success, or a negative errno code otherwise.
	 */
	int (*begin_fb_access)(struct drm_plane *plane, struct drm_plane_state *new_plane_state);

	/**
	 * @end_fb_access:
	 *
	 * This hook cleans up resources allocated by @begin_fb_access. It it called
	 * at the end of a commit for the new plane state.
	 */
	void (*end_fb_access)(struct drm_plane *plane, struct drm_plane_state *new_plane_state);

	/**
	 * @atomic_check:
	 *
	 * Drivers should check plane specific constraints in this hook.
	 *
	 * When using drm_atomic_helper_check_planes() plane's @atomic_check
	 * hooks are called before the ones for CRTCs, which allows drivers to
	 * request shared resources that the CRTC controls here. For more
	 * complicated dependencies the driver can call the provided check helpers
	 * multiple times until the computed state has a final configuration and
	 * everything has been checked.
	 *
	 * This function is also allowed to inspect any other object's state and
	 * can add more state objects to the atomic commit if needed. Care must
	 * be taken though to ensure that state check and compute functions for
	 * these added states are all called, and derived state in other objects
	 * all updated. Again the recommendation is to just call check helpers
	 * until a maximal configuration is reached.
	 *
	 * This callback is used by the atomic modeset helpers, but it is
	 * optional.
	 *
	 * NOTE:
	 *
	 * This function is called in the check phase of an atomic update. The
	 * driver is not allowed to change anything outside of the
	 * &drm_atomic_state update tracking structure.
	 *
	 * RETURNS:
	 *
	 * 0 on success, -EINVAL if the state or the transition can't be
	 * supported, -ENOMEM on memory allocation failure and -EDEADLK if an
	 * attempt to obtain another state object ran into a &drm_modeset_lock
	 * deadlock.
	 */
	int (*atomic_check)(struct drm_plane *plane,
			    struct drm_atomic_state *state);

	/**
	 * @atomic_update:
	 *
	 * Drivers should use this function to update the plane state.  This
	 * hook is called in-between the &drm_crtc_helper_funcs.atomic_begin and
	 * drm_crtc_helper_funcs.atomic_flush callbacks.
	 *
	 * Note that the power state of the display pipe when this function is
	 * called depends upon the exact helpers and calling sequence the driver
	 * has picked. See drm_atomic_helper_commit_planes() for a discussion of
	 * the tradeoffs and variants of plane commit helpers.
	 *
	 * This callback is used by the atomic modeset helpers, but it is optional.
	 */
	void (*atomic_update)(struct drm_plane *plane,
			      struct drm_atomic_state *state);

	/**
	 * @atomic_enable:
	 *
	 * Drivers should use this function to unconditionally enable a plane.
	 * This hook is called in-between the &drm_crtc_helper_funcs.atomic_begin
	 * and drm_crtc_helper_funcs.atomic_flush callbacks. It is called after
	 * @atomic_update, which will be called for all enabled planes. Drivers
	 * that use @atomic_enable should set up a plane in @atomic_update and
	 * afterwards enable the plane in @atomic_enable. If a plane needs to be
	 * enabled before installing the scanout buffer, drivers can still do
	 * so in @atomic_update.
	 *
	 * Note that the power state of the display pipe when this function is
	 * called depends upon the exact helpers and calling sequence the driver
	 * has picked. See drm_atomic_helper_commit_planes() for a discussion of
	 * the tradeoffs and variants of plane commit helpers.
	 *
	 * This callback is used by the atomic modeset helpers, but it is
	 * optional. If implemented, @atomic_enable should be the inverse of
	 * @atomic_disable. Drivers that don't want to use either can still
	 * implement the complete plane update in @atomic_update.
	 */
	void (*atomic_enable)(struct drm_plane *plane,
			      struct drm_atomic_state *state);

	/**
	 * @atomic_disable:
	 *
	 * Drivers should use this function to unconditionally disable a plane.
	 * This hook is called in-between the
	 * &drm_crtc_helper_funcs.atomic_begin and
	 * drm_crtc_helper_funcs.atomic_flush callbacks. It is an alternative to
	 * @atomic_update, which will be called for disabling planes, too, if
	 * the @atomic_disable hook isn't implemented.
	 *
	 * This hook is also useful to disable planes in preparation of a modeset,
	 * by calling drm_atomic_helper_disable_planes_on_crtc() from the
	 * &drm_crtc_helper_funcs.disable hook.
	 *
	 * Note that the power state of the display pipe when this function is
	 * called depends upon the exact helpers and calling sequence the driver
	 * has picked. See drm_atomic_helper_commit_planes() for a discussion of
	 * the tradeoffs and variants of plane commit helpers.
	 *
	 * This callback is used by the atomic modeset helpers, but it is
	 * optional. It's intended to reverse the effects of @atomic_enable.
	 */
	void (*atomic_disable)(struct drm_plane *plane,
			       struct drm_atomic_state *state);

	/**
	 * @atomic_async_check:
	 *
	 * Drivers should set this function pointer to check if the plane's
	 * atomic state can be updated in a async fashion. Here async means
	 * "not vblank synchronized".
	 *
	 * This hook is called by drm_atomic_async_check() to establish if a
	 * given update can be committed asynchronously, that is, if it can
	 * jump ahead of the state currently queued for update.
	 *
	 * RETURNS:
	 *
	 * Return 0 on success and any error returned indicates that the update
	 * can not be applied in asynchronous manner.
	 */
	int (*atomic_async_check)(struct drm_plane *plane,
				  struct drm_atomic_state *state);

	/**
	 * @atomic_async_update:
	 *
	 * Drivers should set this function pointer to perform asynchronous
	 * updates of planes, that is, jump ahead of the currently queued
	 * state and update the plane. Here async means "not vblank
	 * synchronized".
	 *
	 * This hook is called by drm_atomic_helper_async_commit().
	 *
	 * An async update will happen on legacy cursor updates. An async
	 * update won't happen if there is an outstanding commit modifying
	 * the same plane.
	 *
	 * When doing async_update drivers shouldn't replace the
	 * &drm_plane_state but update the current one with the new plane
	 * configurations in the new plane_state.
	 *
	 * Drivers should also swap the framebuffers between current plane
	 * state (&drm_plane.state) and new_state.
	 * This is required since cleanup for async commits is performed on
	 * the new state, rather than old state like for traditional commits.
	 * Since we want to give up the reference on the current (old) fb
	 * instead of our brand new one, swap them in the driver during the
	 * async commit.
	 *
	 * FIXME:
	 *  - It only works for single plane updates
	 *  - Async Pageflips are not supported yet
	 *  - Some hw might still scan out the old buffer until the next
	 *    vblank, however we let go of the fb references as soon as
	 *    we run this hook. For now drivers must implement their own workers
	 *    for deferring if needed, until a common solution is created.
	 */
	void (*atomic_async_update)(struct drm_plane *plane,
				    struct drm_atomic_state *state);
};
```

**format modifier**： 用于描述在特定硬件上如何存储图像数据，通常用于优化图像数据的存储布局，以提高性能，减少内存带宽消耗，使图像数据能够高效的传输和渲染。

在drm框架中，format modifier通过`DRM_FORMAT_MOD_*`宏来定义，用于表示各种不同的存储布局方式：

+ DRM_FORMAT_MOD_LINEAR: 线性布局，这是最常见的存储方式，图像数据按行存储。

```c
#define fourcc_mod_code(vendor, val) \
	((((__u64)DRM_FORMAT_MOD_VENDOR_## vendor) << 56) | ((val) & 0x00ffffffffffffffULL))
#define DRM_FORMAT_MOD_LINEAR	fourcc_mod_code(NONE, 0)
```

### 2.4 struct drm_crtc

`drm_crtc`抽象了显示控制器，CRTC (Cathode Ray Tube Controller)，使用`drm_crtc_init_with_planes()`进行初始化，初始化后链接到`drm_mode_config.crtc_list`

```c
// include/drm/drm_crtc.h
/**
 * struct drm_crtc - central CRTC control structure
 *
 * Each CRTC may have one or more connectors associated with it.  This structure
 * allows the CRTC to be controlled.
 */
struct drm_crtc {
	/** @dev: parent DRM device */
	// 指向drm_device
	struct drm_device *dev;
	/** @port: OF node used by drm_of_find_possible_crtcs(). */
	struct device_node *port;
	/**
	 * @head:
	 *
	 * List of all CRTCs on @dev, linked from &drm_mode_config.crtc_list.
	 * Invariant over the lifetime of @dev and therefore does not need
	 * locking.
	 */
	// 链接到drm_device->drm_mode_config->crtc_list链表
	struct list_head head;

	/** @name: human readable name, can be overwritten by the driver */
	// crtc名称，Phytium DC驱动里面CRTC的名字是phys_pipe 0和phys_pipe 1
	char *name;

	/**
	 * @mutex:
	 *
	 * This provides a read lock for the overall CRTC state (mode, dpms
	 * state, ...) and a write lock for everything which can be update
	 * without a full modeset (fb, cursor data, CRTC properties ...). A full
	 * modeset also need to grab &drm_mode_config.connection_mutex.
	 *
	 * For atomic drivers specifically this protects @state.
	 */
	struct drm_modeset_lock mutex;

	/** @base: base KMS object for ID tracking etc. */
	struct drm_mode_object base;

	/**
	 * @primary:
	 * Primary plane for this CRTC. Note that this is only
	 * relevant for legacy IOCTL, it specifies the plane implicitly used by
	 * the SETCRTC and PAGE_FLIP IOCTLs. It does not have any significance
	 * beyond that.
	 */
	// 指向主图层
	struct drm_plane *primary;

	/**
	 * @cursor:
	 * Cursor plane for this CRTC. Note that this is only relevant for
	 * legacy IOCTL, it specifies the plane implicitly used by the SETCURSOR
	 * and SETCURSOR2 IOCTLs. It does not have any significance
	 * beyond that.
	 */
	// 指向光标图层
	struct drm_plane *cursor;

	/**
	 * @index: Position inside the mode_config.list, can be used as an array
	 * index. It is invariant over the lifetime of the CRTC.
	 */
	unsigned index;

	/**
	 * @cursor_x: Current x position of the cursor, used for universal
	 * cursor planes because the SETCURSOR IOCTL only can update the
	 * framebuffer without supplying the coordinates. Drivers should not use
	 * this directly, atomic drivers should look at &drm_plane_state.crtc_x
	 * of the cursor plane instead.
	 */
	int cursor_x;
	/**
	 * @cursor_y: Current y position of the cursor, used for universal
	 * cursor planes because the SETCURSOR IOCTL only can update the
	 * framebuffer without supplying the coordinates. Drivers should not use
	 * this directly, atomic drivers should look at &drm_plane_state.crtc_y
	 * of the cursor plane instead.
	 */
	int cursor_y;

	/**
	 * @enabled:
	 *
	 * Is this CRTC enabled? Should only be used by legacy drivers, atomic
	 * drivers should instead consult &drm_crtc_state.enable and
	 * &drm_crtc_state.active. Atomic drivers can update this by calling
	 * drm_atomic_helper_update_legacy_modeset_state().
	 */
	bool enabled;

	/**
	 * @mode:
	 *
	 * Current mode timings. Should only be used by legacy drivers, atomic
	 * drivers should instead consult &drm_crtc_state.mode. Atomic drivers
	 * can update this by calling
	 * drm_atomic_helper_update_legacy_modeset_state().
	 */
	struct drm_display_mode mode;

	/**
	 * @hwmode:
	 *
	 * Programmed mode in hw, after adjustments for encoders, crtc, panel
	 * scaling etc. Should only be used by legacy drivers, for high
	 * precision vblank timestamps in
	 * drm_crtc_vblank_helper_get_vblank_timestamp().
	 *
	 * Note that atomic drivers should not use this, but instead use
	 * &drm_crtc_state.adjusted_mode. And for high-precision timestamps
	 * drm_crtc_vblank_helper_get_vblank_timestamp() used
	 * &drm_vblank_crtc.hwmode,
	 * which is filled out by calling drm_calc_timestamping_constants().
	 */
	struct drm_display_mode hwmode;

	/**
	 * @x:
	 * x position on screen. Should only be used by legacy drivers, atomic
	 * drivers should look at &drm_plane_state.crtc_x of the primary plane
	 * instead. Updated by calling
	 * drm_atomic_helper_update_legacy_modeset_state().
	 */
	int x;
	/**
	 * @y:
	 * y position on screen. Should only be used by legacy drivers, atomic
	 * drivers should look at &drm_plane_state.crtc_y of the primary plane
	 * instead. Updated by calling
	 * drm_atomic_helper_update_legacy_modeset_state().
	 */
	int y;

	/** @funcs: CRTC control functions */
	// CRTC控制的相关接口，由厂家实现
	const struct drm_crtc_funcs *funcs;

	/**
	 * @gamma_size: Size of legacy gamma ramp reported to userspace. Set up
	 * by calling drm_mode_crtc_set_gamma_size().
	 */
	uint32_t gamma_size;

	/**
	 * @gamma_store: Gamma ramp values used by the legacy SETGAMMA and
	 * GETGAMMA IOCTls. Set up by calling drm_mode_crtc_set_gamma_size().
	 */
	uint16_t *gamma_store;

	/** @helper_private: mid-layer private data */
	const struct drm_crtc_helper_funcs *helper_private;

	/** @properties: property tracking for this CRTC */
	struct drm_object_properties properties;

	/**
	 * @state:
	 *
	 * Current atomic state for this CRTC.
	 *
	 * This is protected by @mutex. Note that nonblocking atomic commits
	 * access the current CRTC state without taking locks. Either by going
	 * through the &struct drm_atomic_state pointers, see
	 * for_each_oldnew_crtc_in_state(), for_each_old_crtc_in_state() and
	 * for_each_new_crtc_in_state(). Or through careful ordering of atomic
	 * commit operations as implemented in the atomic helpers, see
	 * &struct drm_crtc_commit.
	 */
	struct drm_crtc_state *state;

	/**
	 * @commit_list:
	 *
	 * List of &drm_crtc_commit structures tracking pending commits.
	 * Protected by @commit_lock. This list holds its own full reference,
	 * as does the ongoing commit.
	 *
	 * "Note that the commit for a state change is also tracked in
	 * &drm_crtc_state.commit. For accessing the immediately preceding
	 * commit in an atomic update it is recommended to just use that
	 * pointer in the old CRTC state, since accessing that doesn't need
	 * any locking or list-walking. @commit_list should only be used to
	 * stall for framebuffer cleanup that's signalled through
	 * &drm_crtc_commit.cleanup_done."
	 */
	struct list_head commit_list;

	/**
	 * @commit_lock:
	 *
	 * Spinlock to protect @commit_list.
	 */
	spinlock_t commit_lock;

#ifdef CONFIG_DEBUG_FS
	/**
	 * @debugfs_entry:
	 *
	 * Debugfs directory for this CRTC.
	 */
	struct dentry *debugfs_entry;
#endif

	/**
	 * @crc:
	 *
	 * Configuration settings of CRC capture.
	 */
	struct drm_crtc_crc crc;

	/**
	 * @fence_context:
	 *
	 * timeline context used for fence operations.
	 */
	unsigned int fence_context;

	/**
	 * @fence_lock:
	 *
	 * spinlock to protect the fences in the fence_context.
	 */
	spinlock_t fence_lock;
	/**
	 * @fence_seqno:
	 *
	 * Seqno variable used as monotonic counter for the fences
	 * created on the CRTC's timeline.
	 */
	unsigned long fence_seqno;

	/**
	 * @timeline_name:
	 *
	 * The name of the CRTC's fence timeline.
	 */
	char timeline_name[32];

	/**
	 * @self_refresh_data: Holds the state for the self refresh helpers
	 *
	 * Initialized via drm_self_refresh_helper_init().
	 */
	struct drm_self_refresh_data *self_refresh_data;
};
```

### struct drm_crtc_state

`struct drm_crtc_state`用来记录当前crtc的显示模式相关的参数

```c
// include/drm/drm_crtc.h
/**
 * struct drm_crtc_state - mutable CRTC state
 *
 * Note that the distinction between @enable and @active is rather subtle:
 * Flipping @active while @enable is set without changing anything else may
 * never return in a failure from the &drm_mode_config_funcs.atomic_check
 * callback. Userspace assumes that a DPMS On will always succeed. In other
 * words: @enable controls resource assignment, @active controls the actual
 * hardware state.
 *
 * The three booleans active_changed, connectors_changed and mode_changed are
 * intended to indicate whether a full modeset is needed, rather than strictly
 * describing what has changed in a commit. See also:
 * drm_atomic_crtc_needs_modeset()
 */
struct drm_crtc_state {
	/** @crtc: backpointer to the CRTC */
	// 指向对应的crtc
	struct drm_crtc *crtc;

	/**
	 * @enable: Whether the CRTC should be enabled, gates all other state.
	 * This controls reservations of shared resources. Actual hardware state
	 * is controlled by @active.
	 */
	// 当前crtc是否启用
	bool enable;

	/**
	 * @active: Whether the CRTC is actively displaying (used for DPMS).
	 * Implies that @enable is set. The driver must not release any shared
	 * resources if @active is set to false but @enable still true, because
	 * userspace expects that a DPMS ON always succeeds.
	 *
	 * Hence drivers must not consult @active in their various
	 * &drm_mode_config_funcs.atomic_check callback to reject an atomic
	 * commit. They can consult it to aid in the computation of derived
	 * hardware state, since even in the DPMS OFF state the display hardware
	 * should be as much powered down as when the CRTC is completely
	 * disabled through setting @enable to false.
	 */
	bool active;

	/**
	 * @planes_changed: Planes on this crtc are updated. Used by the atomic
	 * helpers and drivers to steer the atomic commit control flow.
	 */
	bool planes_changed : 1;

	/**
	 * @mode_changed: @mode or @enable has been changed. Used by the atomic
	 * helpers and drivers to steer the atomic commit control flow. See also
	 * drm_atomic_crtc_needs_modeset().
	 *
	 * Drivers are supposed to set this for any CRTC state changes that
	 * require a full modeset. They can also reset it to false if e.g. a
	 * @mode change can be done without a full modeset by only changing
	 * scaler settings.
	 */
	bool mode_changed : 1;

	/**
	 * @active_changed: @active has been toggled. Used by the atomic
	 * helpers and drivers to steer the atomic commit control flow. See also
	 * drm_atomic_crtc_needs_modeset().
	 */
	bool active_changed : 1;

	/**
	 * @connectors_changed: Connectors to this crtc have been updated,
	 * either in their state or routing. Used by the atomic
	 * helpers and drivers to steer the atomic commit control flow. See also
	 * drm_atomic_crtc_needs_modeset().
	 *
	 * Drivers are supposed to set this as-needed from their own atomic
	 * check code, e.g. from &drm_encoder_helper_funcs.atomic_check
	 */
	bool connectors_changed : 1;
	/**
	 * @zpos_changed: zpos values of planes on this crtc have been updated.
	 * Used by the atomic helpers and drivers to steer the atomic commit
	 * control flow.
	 */
	bool zpos_changed : 1;
	/**
	 * @color_mgmt_changed: Color management properties have changed
	 * (@gamma_lut, @degamma_lut or @ctm). Used by the atomic helpers and
	 * drivers to steer the atomic commit control flow.
	 */
	bool color_mgmt_changed : 1;

	/**
	 * @no_vblank:
	 *
	 * Reflects the ability of a CRTC to send VBLANK events. This state
	 * usually depends on the pipeline configuration. If set to true, DRM
	 * atomic helpers will send out a fake VBLANK event during display
	 * updates after all hardware changes have been committed. This is
	 * implemented in drm_atomic_helper_fake_vblank().
	 *
	 * One usage is for drivers and/or hardware without support for VBLANK
	 * interrupts. Such drivers typically do not initialize vblanking
	 * (i.e., call drm_vblank_init() with the number of CRTCs). For CRTCs
	 * without initialized vblanking, this field is set to true in
	 * drm_atomic_helper_check_modeset(), and a fake VBLANK event will be
	 * send out on each update of the display pipeline by
	 * drm_atomic_helper_fake_vblank().
	 *
	 * Another usage is CRTCs feeding a writeback connector operating in
	 * oneshot mode. In this case the fake VBLANK event is only generated
	 * when a job is queued to the writeback connector, and we want the
	 * core to fake VBLANK events when this part of the pipeline hasn't
	 * changed but others had or when the CRTC and connectors are being
	 * disabled.
	 *
	 * __drm_atomic_helper_crtc_duplicate_state() will not reset the value
	 * from the current state, the CRTC driver is then responsible for
	 * updating this field when needed.
	 *
	 * Note that the combination of &drm_crtc_state.event == NULL and
	 * &drm_crtc_state.no_blank == true is valid and usually used when the
	 * writeback connector attached to the CRTC has a new job queued. In
	 * this case the driver will send the VBLANK event on its own when the
	 * writeback job is complete.
	 */
	bool no_vblank : 1;

	/**
	 * @plane_mask: Bitmask of drm_plane_mask(plane) of planes attached to
	 * this CRTC.
	 */
	// 记录当前crct对应的plane
	u32 plane_mask;

	/**
	 * @connector_mask: Bitmask of drm_connector_mask(connector) of
	 * connectors attached to this CRTC.
	 */
	u32 connector_mask;

	/**
	 * @encoder_mask: Bitmask of drm_encoder_mask(encoder) of encoders
	 * attached to this CRTC.
	 */
	u32 encoder_mask;

	/**
	 * @adjusted_mode:
	 *
	 * Internal display timings which can be used by the driver to handle
	 * differences between the mode requested by userspace in @mode and what
	 * is actually programmed into the hardware.
	 *
	 * For drivers using &drm_bridge, this stores hardware display timings
	 * used between the CRTC and the first bridge. For other drivers, the
	 * meaning of the adjusted_mode field is purely driver implementation
	 * defined information, and will usually be used to store the hardware
	 * display timings used between the CRTC and encoder blocks.
	 */
	struct drm_display_mode adjusted_mode;

	/**
	 * @mode:
	 *
	 * Display timings requested by userspace. The driver should try to
	 * match the refresh rate as close as possible (but note that it's
	 * undefined what exactly is close enough, e.g. some of the HDMI modes
	 * only differ in less than 1% of the refresh rate). The active width
	 * and height as observed by userspace for positioning planes must match
	 * exactly.
	 *
	 * For external connectors where the sink isn't fixed (like with a
	 * built-in panel), this mode here should match the physical mode on the
	 * wire to the last details (i.e. including sync polarities and
	 * everything).
	 */
	struct drm_display_mode mode;

	/**
	 * @mode_blob: &drm_property_blob for @mode, for exposing the mode to
	 * atomic userspace.
	 */
	struct drm_property_blob *mode_blob;

	/**
	 * @degamma_lut:
	 *
	 * Lookup table for converting framebuffer pixel data before apply the
	 * color conversion matrix @ctm. See drm_crtc_enable_color_mgmt(). The
	 * blob (if not NULL) is an array of &struct drm_color_lut.
	 */
	struct drm_property_blob *degamma_lut;

	/**
	 * @ctm:
	 *
	 * Color transformation matrix. See drm_crtc_enable_color_mgmt(). The
	 * blob (if not NULL) is a &struct drm_color_ctm.
	 */
	struct drm_property_blob *ctm;

	/**
	 * @gamma_lut:
	 *
	 * Lookup table for converting pixel data after the color conversion
	 * matrix @ctm.  See drm_crtc_enable_color_mgmt(). The blob (if not
	 * NULL) is an array of &struct drm_color_lut.
	 *
	 * Note that for mostly historical reasons stemming from Xorg heritage,
	 * this is also used to store the color map (also sometimes color lut,
	 * CLUT or color palette) for indexed formats like DRM_FORMAT_C8.
	 */
	struct drm_property_blob *gamma_lut;

	/**
	 * @target_vblank:
	 *
	 * Target vertical blank period when a page flip
	 * should take effect.
	 */
	u32 target_vblank;

	/**
	 * @async_flip:
	 *
	 * This is set when DRM_MODE_PAGE_FLIP_ASYNC is set in the legacy
	 * PAGE_FLIP IOCTL. It's not wired up for the atomic IOCTL itself yet.
	 */
	bool async_flip;

	/**
	 * @vrr_enabled:
	 *
	 * Indicates if variable refresh rate should be enabled for the CRTC.
	 * Support for the requested vrr state will depend on driver and
	 * hardware capabiltiy - lacking support is not treated as failure.
	 */
	bool vrr_enabled;

	/**
	 * @self_refresh_active:
	 *
	 * Used by the self refresh helpers to denote when a self refresh
	 * transition is occurring. This will be set on enable/disable callbacks
	 * when self refresh is being enabled or disabled. In some cases, it may
	 * not be desirable to fully shut off the crtc during self refresh.
	 * CRTC's can inspect this flag and determine the best course of action.
	 */
	bool self_refresh_active;

	/**
	 * @scaling_filter:
	 *
	 * Scaling filter to be applied
	 */
	enum drm_scaling_filter scaling_filter;

	/**
	 * @event:
	 *
	 * Optional pointer to a DRM event to signal upon completion of the
	 * state update. The driver must send out the event when the atomic
	 * commit operation completes. There are two cases:
	 *
	 *  - The event is for a CRTC which is being disabled through this
	 *    atomic commit. In that case the event can be send out any time
	 *    after the hardware has stopped scanning out the current
	 *    framebuffers. It should contain the timestamp and counter for the
	 *    last vblank before the display pipeline was shut off. The simplest
	 *    way to achieve that is calling drm_crtc_send_vblank_event()
	 *    somewhen after drm_crtc_vblank_off() has been called.
	 *
	 *  - For a CRTC which is enabled at the end of the commit (even when it
	 *    undergoes an full modeset) the vblank timestamp and counter must
	 *    be for the vblank right before the first frame that scans out the
	 *    new set of buffers. Again the event can only be sent out after the
	 *    hardware has stopped scanning out the old buffers.
	 *
	 *  - Events for disabled CRTCs are not allowed, and drivers can ignore
	 *    that case.
	 *
	 * For very simple hardware without VBLANK interrupt, enabling
	 * &struct drm_crtc_state.no_vblank makes DRM's atomic commit helpers
	 * send a fake VBLANK event at the end of the display update after all
	 * hardware changes have been applied. See
	 * drm_atomic_helper_fake_vblank().
	 *
	 * For more complex hardware this
	 * can be handled by the drm_crtc_send_vblank_event() function,
	 * which the driver should call on the provided event upon completion of
	 * the atomic commit. Note that if the driver supports vblank signalling
	 * and timestamping the vblank counters and timestamps must agree with
	 * the ones returned from page flip events. With the current vblank
	 * helper infrastructure this can be achieved by holding a vblank
	 * reference while the page flip is pending, acquired through
	 * drm_crtc_vblank_get() and released with drm_crtc_vblank_put().
	 * Drivers are free to implement their own vblank counter and timestamp
	 * tracking though, e.g. if they have accurate timestamp registers in
	 * hardware.
	 *
	 * For hardware which supports some means to synchronize vblank
	 * interrupt delivery with committing display state there's also
	 * drm_crtc_arm_vblank_event(). See the documentation of that function
	 * for a detailed discussion of the constraints it needs to be used
	 * safely.
	 *
	 * If the device can't notify of flip completion in a race-free way
	 * at all, then the event should be armed just after the page flip is
	 * committed. In the worst case the driver will send the event to
	 * userspace one frame too late. This doesn't allow for a real atomic
	 * update, but it should avoid tearing.
	 */
	struct drm_pending_vblank_event *event;

	/**
	 * @commit:
	 *
	 * This tracks how the commit for this update proceeds through the
	 * various phases. This is never cleared, except when we destroy the
	 * state, so that subsequent commits can synchronize with previous ones.
	 */
	struct drm_crtc_commit *commit;

	/** @state: backpointer to global drm_atomic_state */
	// 指向drm_atomic_state
	struct drm_atomic_state *state;
};
```

### 2.5 struct drm_device

linux内核使用`struct drm_device`数据结构来描述一个`drm`设备

```c
// include/drm/drm_device.h
/**
 * struct drm_device - DRM device structure
 *
 * This structure represent a complete card that
 * may contain multiple heads.
 */
struct drm_device {
	/** @if_version: Highest interface version set */
	int if_version;

	/** @ref: Object ref-count */
	/* 引用计数，使用drm_dev_get()和drm_dev_put()获取和释放引用计数 */
	struct kref ref;

	/** @dev: Device structure of bus-device */
	/* 设备驱动模型中的device，可以将drm_device看做其子类 */
	struct device *dev;

	// 使用managed来管理drm驱动初始化过程中分配的内存资源
	/**
	 * @managed:
	 *
	 * Managed resources linked to the lifetime of this &drm_device as
	 * tracked by @ref.
	 */
	struct {
		/** @managed.resources: managed resources list */
		// resources 会将所有的资源链接在一起
		struct list_head resources;
		/** @managed.final_kfree: pointer for final kfree() call */
		void *final_kfree;
		// 操作managed时的锁
		/** @managed.lock: protects @managed.resources */
		spinlock_t lock;
	} managed;

	/** @driver: DRM driver managing the device */
	/* 指向drm驱动 */
	const struct drm_driver *driver;

	/**
	 * @dev_private:
	 *
	 * DRM driver private data. This is deprecated and should be left set to
	 * NULL.
	 *
	 * Instead of using this pointer it is recommended that drivers use
	 * devm_drm_dev_alloc() and embed struct &drm_device in their larger
	 * per-device structure.
	 */
	void *dev_private;

	/**
	 * @primary:
	 *
	 * Primary node. Drivers should not interact with this
	 * directly. debugfs interfaces can be registered with
	 * drm_debugfs_add_file(), and sysfs should be directly added on the
	 * hardware (and not character device node) struct device @dev.
	 */
	struct drm_minor *primary;

	/**
	 * @render:
	 *
	 * Render node. Drivers should not interact with this directly ever.
	 * Drivers should not expose any additional interfaces in debugfs or
	 * sysfs on this node.
	 */
	struct drm_minor *render;

	/** @accel: Compute Acceleration node */
	struct drm_minor *accel;

	/**
	 * @registered:
	 *
	 * Internally used by drm_dev_register() and drm_connector_register().
	 */
	/* 设备是否已注册 */
	bool registered;

	/**
	 * @master:
	 *
	 * Currently active master for this device.
	 * Protected by &master_mutex
	 */
	struct drm_master *master;

	/**
	 * @driver_features: per-device driver features
	 *
	 * Drivers can clear specific flags here to disallow
	 * certain features on a per-device basis while still
	 * sharing a single &struct drm_driver instance across
	 * all devices.
	 */
	// drm驱动特性
	u32 driver_features;

	/**
	 * @unplugged:
	 *
	 * Flag to tell if the device has been unplugged.
	 * See drm_dev_enter() and drm_dev_is_unplugged().
	 */
	bool unplugged;

	/** @anon_inode: inode for private address-space */
	struct inode *anon_inode;

	/** @unique: Unique name of the device */
	/* drm设备的名称 */
	char *unique;

	/**
	 * @struct_mutex:
	 *
	 * Lock for others (not &drm_minor.master and &drm_file.is_master)
	 *
	 * WARNING:
	 * Only drivers annotated with DRIVER_LEGACY should be using this.
	 */
	struct mutex struct_mutex;

	/**
	 * @master_mutex:
	 *
	 * Lock for &drm_minor.master and &drm_file.is_master
	 */
	struct mutex master_mutex;

	/**
	 * @open_count:
	 *
	 * Usage counter for outstanding files open,
	 * protected by drm_global_mutex
	 */
	atomic_t open_count;

	/** @filelist_mutex: Protects @filelist. */
	struct mutex filelist_mutex;
	/**
	 * @filelist:
	 *
	 * List of userspace clients, linked through &drm_file.lhead.
	 */
	struct list_head filelist;

	/**
	 * @filelist_internal:
	 *
	 * List of open DRM files for in-kernel clients.
	 * Protected by &filelist_mutex.
	 */
	// 链接kernel中创建的drm_file
	struct list_head filelist_internal;

	/**
	 * @clientlist_mutex:
	 *
	 * Protects &clientlist access.
	 */
	struct mutex clientlist_mutex;

	/**
	 * @clientlist:
	 *
	 * List of in-kernel clients. Protected by &clientlist_mutex.
	 */
	struct list_head clientlist;

	/**
	 * @vblank_disable_immediate:
	 *
	 * If true, vblank interrupt will be disabled immediately when the
	 * refcount drops to zero, as opposed to via the vblank disable
	 * timer.
	 *
	 * This can be set to true it the hardware has a working vblank counter
	 * with high-precision timestamping (otherwise there are races) and the
	 * driver uses drm_crtc_vblank_on() and drm_crtc_vblank_off()
	 * appropriately. See also @max_vblank_count and
	 * &drm_crtc_funcs.get_vblank_counter.
	 */
	bool vblank_disable_immediate;

	/**
	 * @vblank:
	 *
	 * Array of vblank tracking structures, one per &struct drm_crtc. For
	 * historical reasons (vblank support predates kernel modesetting) this
	 * is free-standing and not part of &struct drm_crtc itself. It must be
	 * initialized explicitly by calling drm_vblank_init().
	 */
	struct drm_vblank_crtc *vblank;

	/**
	 * @vblank_time_lock:
	 *
	 *  Protects vblank count and time updates during vblank enable/disable
	 */
	spinlock_t vblank_time_lock;
	/**
	 * @vbl_lock: Top-level vblank references lock, wraps the low-level
	 * @vblank_time_lock.
	 */
	spinlock_t vbl_lock;

	/**
	 * @max_vblank_count:
	 *
	 * Maximum value of the vblank registers. This value +1 will result in a
	 * wrap-around of the vblank register. It is used by the vblank core to
	 * handle wrap-arounds.
	 *
	 * If set to zero the vblank core will try to guess the elapsed vblanks
	 * between times when the vblank interrupt is disabled through
	 * high-precision timestamps. That approach is suffering from small
	 * races and imprecision over longer time periods, hence exposing a
	 * hardware vblank counter is always recommended.
	 *
	 * This is the statically configured device wide maximum. The driver
	 * can instead choose to use a runtime configurable per-crtc value
	 * &drm_vblank_crtc.max_vblank_count, in which case @max_vblank_count
	 * must be left at zero. See drm_crtc_set_max_vblank_count() on how
	 * to use the per-crtc value.
	 *
	 * If non-zero, &drm_crtc_funcs.get_vblank_counter must be set.
	 */
	u32 max_vblank_count;

	/** @vblank_event_list: List of vblank events */
	struct list_head vblank_event_list;

	/**
	 * @event_lock:
	 *
	 * Protects @vblank_event_list and event delivery in
	 * general. See drm_send_event() and drm_send_event_locked().
	 */
	spinlock_t event_lock;

	/** @num_crtcs: Number of CRTCs on this device */
	/* CRTC的数量 */
	unsigned int num_crtcs;

	/** @mode_config: Current mode config */
	struct drm_mode_config mode_config;

	/** @object_name_lock: GEM information */
	struct mutex object_name_lock;

	/** @object_name_idr: GEM information */
	struct idr object_name_idr;

	/** @vma_offset_manager: GEM information */
	struct drm_vma_offset_manager *vma_offset_manager;

	/** @vram_mm: VRAM MM memory manager */
	struct drm_vram_mm *vram_mm;

	/**
	 * @switch_power_state:
	 *
	 * Power state of the client.
	 * Used by drivers supporting the switcheroo driver.
	 * The state is maintained in the
	 * &vga_switcheroo_client_ops.set_gpu_state callback
	 */
	enum switch_power_state switch_power_state;

	/**
	 * @fb_helper:
	 *
	 * Pointer to the fbdev emulation structure.
	 * Set by drm_fb_helper_init() and cleared by drm_fb_helper_fini().
	 */
	struct drm_fb_helper *fb_helper;

	/**
	 * @debugfs_mutex:
	 *
	 * Protects &debugfs_list access.
	 */
	struct mutex debugfs_mutex;

	/**
	 * @debugfs_list:
	 *
	 * List of debugfs files to be created by the DRM device. The files
	 * must be added during drm_dev_register().
	 */
	/* 保存 struct drm_debugfs_entry 的链表 */
	struct list_head debugfs_list;

	/* Everything below here is for legacy driver, never use! */
	/* private: */
#if IS_ENABLED(CONFIG_DRM_LEGACY)
	/* List of devices per driver for stealth attach cleanup */
	struct list_head legacy_dev_list;

#ifdef __alpha__
	/** @hose: PCI hose, only used on ALPHA platforms. */
	struct pci_controller *hose;
#endif

	/* AGP data */
	struct drm_agp_head *agp;

	/* Context handle management - linked list of context handles */
	struct list_head ctxlist;

	/* Context handle management - mutex for &ctxlist */
	struct mutex ctxlist_mutex;

	/* Context handle management */
	struct idr ctx_idr;

	/* Memory management - linked list of regions */
	struct list_head maplist;

	/* Memory management - user token hash table for maps */
	struct drm_open_hash map_hash;

	/* Context handle management - list of vmas (for debugging) */
	struct list_head vmalist;

	/* Optional pointer for DMA support */
	struct drm_device_dma *dma;

	/* Context swapping flag */
	__volatile__ long context_flag;

	/* Last current context */
	int last_context;

	/* Lock for &buf_use and a few other things. */
	spinlock_t buf_lock;

	/* Usage counter for buffers in use -- cannot alloc */
	int buf_use;

	/* Buffer allocation in progress */
	atomic_t buf_alloc;

	struct {
		int context;
		struct drm_hw_lock *lock;
	} sigdata;

	struct drm_local_map *agp_buffer_map;
	unsigned int agp_buffer_token;

	/* Scatter gather memory */
	struct drm_sg_mem *sg;

	/* IRQs */
	bool irq_enabled;
	int irq;
#endif
};
```

### 2.6 struct drm_mode_config

linux内核使用`struct drm_mode_config`来描述显示模式配置信息，`drm_mode_config`的主要功能之一是提供对显示器模式的管理和配置，包括添加、删除、修改、查询显示器模式的能力，在整个驱动的初始化过程中`struct drm_mode_config`会记录crtc plane等的信息。`drm_device->mode_config`存储其信息，在`drm_mode_config_init()`中进行初始化

```c
/**
 * struct drm_mode_config - Mode configuration control structure
 * @min_width: minimum fb pixel width on this device
 * @min_height: minimum fb pixel height on this device
 * @max_width: maximum fb pixel width on this device
 * @max_height: maximum fb pixel height on this device
 * @funcs: core driver provided mode setting functions
 * @poll_enabled: track polling support for this device
 * @poll_running: track polling status for this device
 * @delayed_event: track delayed poll uevent deliver for this device
 * @output_poll_work: delayed work for polling in process context
 * @preferred_depth: preferred RBG pixel depth, used by fb helpers
 * @prefer_shadow: hint to userspace to prefer shadow-fb rendering
 * @cursor_width: hint to userspace for max cursor width
 * @cursor_height: hint to userspace for max cursor height
 * @helper_private: mid-layer private data
 *
 * Core mode resource tracking structure.  All CRTC, encoders, and connectors
 * enumerated by the driver are added here, as are global properties.  Some
 * global restrictions are also here, e.g. dimension restrictions.
 *
 * Framebuffer sizes refer to the virtual screen that can be displayed by
 * the CRTC. This can be different from the physical resolution programmed.
 * The minimum width and height, stored in @min_width and @min_height,
 * describe the smallest size of the framebuffer. It correlates to the
 * minimum programmable resolution.
 * The maximum width, stored in @max_width, is typically limited by the
 * maximum pitch between two adjacent scanlines. The maximum height, stored
 * in @max_height, is usually only limited by the amount of addressable video
 * memory. For hardware that has no real maximum, drivers should pick a
 * reasonable default.
 *
 * See also @DRM_SHADOW_PLANE_MAX_WIDTH and @DRM_SHADOW_PLANE_MAX_HEIGHT.
 */
struct drm_mode_config {
	/**
	 * @mutex:
	 *
	 * This is the big scary modeset BKL which protects everything that
	 * isn't protect otherwise. Scope is unclear and fuzzy, try to remove
	 * anything from under its protection and move it into more well-scoped
	 * locks.
	 *
	 * The one important thing this protects is the use of @acquire_ctx.
	 */
	struct mutex mutex;

	/**
	 * @connection_mutex:
	 *
	 * This protects connector state and the connector to encoder to CRTC
	 * routing chain.
	 *
	 * For atomic drivers specifically this protects &drm_connector.state.
	 */
	struct drm_modeset_lock connection_mutex;

	/**
	 * @acquire_ctx:
	 *
	 * Global implicit acquire context used by atomic drivers for legacy
	 * IOCTLs. Deprecated, since implicit locking contexts make it
	 * impossible to use driver-private &struct drm_modeset_lock. Users of
	 * this must hold @mutex.
	 */
	struct drm_modeset_acquire_ctx *acquire_ctx;

	/**
	 * @idr_mutex:
	 *
	 * Mutex for KMS ID allocation and management. Protects both @object_idr
	 * and @tile_idr.
	 */
	struct mutex idr_mutex;

	/**
	 * @object_idr:
	 *
	 * Main KMS ID tracking object. Use this idr for all IDs, fb, crtc,
	 * connector, modes - just makes life easier to have only one.
	 */
	// 为framebuffer crtc plane分配唯一id，并管理它们
	struct idr object_idr;

	/**
	 * @tile_idr:
	 *
	 * Use this idr for allocating new IDs for tiled sinks like use in some
	 * high-res DP MST screens.
	 */
	struct idr tile_idr;

	/** @fb_lock: Mutex to protect fb the global @fb_list and @num_fb. */
	struct mutex fb_lock;
	/** @num_fb: Number of entries on @fb_list. */
	/* drm_framebuffer的个数 */
	int num_fb;
	/** @fb_list: List of all &struct drm_framebuffer. */
	/* 链接所有的struct drm_framebuffer */
	struct list_head fb_list;

	/**
	 * @connector_list_lock: Protects @num_connector and
	 * @connector_list and @connector_free_list.
	 */
	spinlock_t connector_list_lock;
	/**
	 * @num_connector: Number of connectors on this device. Protected by
	 * @connector_list_lock.
	 */
	int num_connector;
	/**
	 * @connector_ida: ID allocator for connector indices.
	 */
	struct ida connector_ida;
	/**
	 * @connector_list:
	 *
	 * List of connector objects linked with &drm_connector.head. Protected
	 * by @connector_list_lock. Only use drm_for_each_connector_iter() and
	 * &struct drm_connector_list_iter to walk this list.
	 */
	/* 链接所有的drm_connector */
	struct list_head connector_list;
	/**
	 * @connector_free_list:
	 *
	 * List of connector objects linked with &drm_connector.free_head.
	 * Protected by @connector_list_lock. Used by
	 * drm_for_each_connector_iter() and
	 * &struct drm_connector_list_iter to savely free connectors using
	 * @connector_free_work.
	 */
	struct llist_head connector_free_list;
	/**
	 * @connector_free_work: Work to clean up @connector_free_list.
	 */
	struct work_struct connector_free_work;

	/**
	 * @num_encoder:
	 *
	 * Number of encoders on this device. This is invariant over the
	 * lifetime of a device and hence doesn't need any locks.
	 */
	/* encoder的个数 */
	int num_encoder;
	/**
	 * @encoder_list:
	 *
	 * List of encoder objects linked with &drm_encoder.head. This is
	 * invariant over the lifetime of a device and hence doesn't need any
	 * locks.
	 */
	/* 链接所有的drm_encoder */
	struct list_head encoder_list;

	/**
	 * @num_total_plane:
	 *
	 * Number of universal (i.e. with primary/curso) planes on this device.
	 * This is invariant over the lifetime of a device and hence doesn't
	 * need any locks.
	 */
	/* plane的个数 */
	int num_total_plane;
	/**
	 * @plane_list:
	 *
	 * List of plane objects linked with &drm_plane.head. This is invariant
	 * over the lifetime of a device and hence doesn't need any locks.
	 */
	/* 链接所有的drm_plane */
	struct list_head plane_list;

	/**
	 * @num_crtc:
	 *
	 * Number of CRTCs on this device linked with &drm_crtc.head. This is invariant over the lifetime
	 * of a device and hence doesn't need any locks.
	 */
	/* crtc的个数 */
	int num_crtc;
	/**
	 * @crtc_list:
	 *
	 * List of CRTC objects linked with &drm_crtc.head. This is invariant
	 * over the lifetime of a device and hence doesn't need any locks.
	 */
	/* 链接所有的CRTC */
	struct list_head crtc_list;

	/**
	 * @property_list:
	 *
	 * List of property type objects linked with &drm_property.head. This is
	 * invariant over the lifetime of a device and hence doesn't need any
	 * locks.
	 */
	struct list_head property_list;

	/**
	 * @privobj_list:
	 *
	 * List of private objects linked with &drm_private_obj.head. This is
	 * invariant over the lifetime of a device and hence doesn't need any
	 * locks.
	 */
	struct list_head privobj_list;
	/* 支持最小帧缓冲区的像素宽度和高度 */
	int min_width, min_height;
	/* 支持最大帧缓冲区的像素宽度和高度 */
	int max_width, max_height;
	/* 用于驱动程序向内核注册显示模式配置的回调函数 */
	const struct drm_mode_config_funcs *funcs;

	/* output poll support */
	bool poll_enabled;
	bool poll_running;
	bool delayed_event;
	struct delayed_work output_poll_work;

	/**
	 * @blob_lock:
	 *
	 * Mutex for blob property allocation and management, protects
	 * @property_blob_list and &drm_file.blobs.
	 */
	struct mutex blob_lock;

	/**
	 * @property_blob_list:
	 *
	 * List of all the blob property objects linked with
	 * &drm_property_blob.head. Protected by @blob_lock.
	 */
	struct list_head property_blob_list;

	/* pointers to standard properties */

	/**
	 * @edid_property: Default connector property to hold the EDID of the
	 * currently connected sink, if any.
	 */
	struct drm_property *edid_property;
	/**
	 * @dpms_property: Default connector property to control the
	 * connector's DPMS state.
	 */
	struct drm_property *dpms_property;
	/**
	 * @path_property: Default connector property to hold the DP MST path
	 * for the port.
	 */
	struct drm_property *path_property;
	/**
	 * @tile_property: Default connector property to store the tile
	 * position of a tiled screen, for sinks which need to be driven with
	 * multiple CRTCs.
	 */
	struct drm_property *tile_property;
	/**
	 * @link_status_property: Default connector property for link status
	 * of a connector
	 */
	struct drm_property *link_status_property;
	/**
	 * @plane_type_property: Default plane property to differentiate
	 * CURSOR, PRIMARY and OVERLAY legacy uses of planes.
	 */
	struct drm_property *plane_type_property;
	/**
	 * @prop_src_x: Default atomic plane property for the plane source
	 * position in the connected &drm_framebuffer.
	 */
	struct drm_property *prop_src_x;
	/**
	 * @prop_src_y: Default atomic plane property for the plane source
	 * position in the connected &drm_framebuffer.
	 */
	struct drm_property *prop_src_y;
	/**
	 * @prop_src_w: Default atomic plane property for the plane source
	 * position in the connected &drm_framebuffer.
	 */
	struct drm_property *prop_src_w;
	/**
	 * @prop_src_h: Default atomic plane property for the plane source
	 * position in the connected &drm_framebuffer.
	 */
	struct drm_property *prop_src_h;
	/**
	 * @prop_crtc_x: Default atomic plane property for the plane destination
	 * position in the &drm_crtc is being shown on.
	 */
	struct drm_property *prop_crtc_x;
	/**
	 * @prop_crtc_y: Default atomic plane property for the plane destination
	 * position in the &drm_crtc is being shown on.
	 */
	struct drm_property *prop_crtc_y;
	/**
	 * @prop_crtc_w: Default atomic plane property for the plane destination
	 * position in the &drm_crtc is being shown on.
	 */
	struct drm_property *prop_crtc_w;
	/**
	 * @prop_crtc_h: Default atomic plane property for the plane destination
	 * position in the &drm_crtc is being shown on.
	 */
	struct drm_property *prop_crtc_h;
	/**
	 * @prop_fb_id: Default atomic plane property to specify the
	 * &drm_framebuffer.
	 */
	struct drm_property *prop_fb_id;
	/**
	 * @prop_in_fence_fd: Sync File fd representing the incoming fences
	 * for a Plane.
	 */
	struct drm_property *prop_in_fence_fd;
	/**
	 * @prop_out_fence_ptr: Sync File fd pointer representing the
	 * outgoing fences for a CRTC. Userspace should provide a pointer to a
	 * value of type s32, and then cast that pointer to u64.
	 */
	struct drm_property *prop_out_fence_ptr;
	/**
	 * @prop_crtc_id: Default atomic plane property to specify the
	 * &drm_crtc.
	 */
	struct drm_property *prop_crtc_id;
	/**
	 * @prop_fb_damage_clips: Optional plane property to mark damaged
	 * regions on the plane in framebuffer coordinates of the framebuffer
	 * attached to the plane.
	 *
	 * The layout of blob data is simply an array of &drm_mode_rect. Unlike
	 * plane src coordinates, damage clips are not in 16.16 fixed point.
	 */
	struct drm_property *prop_fb_damage_clips;
	/**
	 * @prop_active: Default atomic CRTC property to control the active
	 * state, which is the simplified implementation for DPMS in atomic
	 * drivers.
	 */
	struct drm_property *prop_active;
	/**
	 * @prop_mode_id: Default atomic CRTC property to set the mode for a
	 * CRTC. A 0 mode implies that the CRTC is entirely disabled - all
	 * connectors must be of and active must be set to disabled, too.
	 */
	struct drm_property *prop_mode_id;
	/**
	 * @prop_vrr_enabled: Default atomic CRTC property to indicate
	 * whether variable refresh rate should be enabled on the CRTC.
	 */
	struct drm_property *prop_vrr_enabled;

	/**
	 * @dvi_i_subconnector_property: Optional DVI-I property to
	 * differentiate between analog or digital mode.
	 */
	struct drm_property *dvi_i_subconnector_property;
	/**
	 * @dvi_i_select_subconnector_property: Optional DVI-I property to
	 * select between analog or digital mode.
	 */
	struct drm_property *dvi_i_select_subconnector_property;

	/**
	 * @dp_subconnector_property: Optional DP property to differentiate
	 * between different DP downstream port types.
	 */
	struct drm_property *dp_subconnector_property;

	/**
	 * @tv_subconnector_property: Optional TV property to differentiate
	 * between different TV connector types.
	 */
	struct drm_property *tv_subconnector_property;
	/**
	 * @tv_select_subconnector_property: Optional TV property to select
	 * between different TV connector types.
	 */
	struct drm_property *tv_select_subconnector_property;

	/**
	 * @legacy_tv_mode_property: Optional TV property to select
	 * the output TV mode.
	 *
	 * Superseded by @tv_mode_property
	 */
	struct drm_property *legacy_tv_mode_property;

	/**
	 * @tv_mode_property: Optional TV property to select the TV
	 * standard output on the connector.
	 */
	struct drm_property *tv_mode_property;

	/**
	 * @tv_left_margin_property: Optional TV property to set the left
	 * margin (expressed in pixels).
	 */
	struct drm_property *tv_left_margin_property;
	/**
	 * @tv_right_margin_property: Optional TV property to set the right
	 * margin (expressed in pixels).
	 */
	struct drm_property *tv_right_margin_property;
	/**
	 * @tv_top_margin_property: Optional TV property to set the right
	 * margin (expressed in pixels).
	 */
	struct drm_property *tv_top_margin_property;
	/**
	 * @tv_bottom_margin_property: Optional TV property to set the right
	 * margin (expressed in pixels).
	 */
	struct drm_property *tv_bottom_margin_property;
	/**
	 * @tv_brightness_property: Optional TV property to set the
	 * brightness.
	 */
	struct drm_property *tv_brightness_property;
	/**
	 * @tv_contrast_property: Optional TV property to set the
	 * contrast.
	 */
	struct drm_property *tv_contrast_property;
	/**
	 * @tv_flicker_reduction_property: Optional TV property to control the
	 * flicker reduction mode.
	 */
	struct drm_property *tv_flicker_reduction_property;
	/**
	 * @tv_overscan_property: Optional TV property to control the overscan
	 * setting.
	 */
	struct drm_property *tv_overscan_property;
	/**
	 * @tv_saturation_property: Optional TV property to set the
	 * saturation.
	 */
	struct drm_property *tv_saturation_property;
	/**
	 * @tv_hue_property: Optional TV property to set the hue.
	 */
	struct drm_property *tv_hue_property;

	/**
	 * @scaling_mode_property: Optional connector property to control the
	 * upscaling, mostly used for built-in panels.
	 */
	struct drm_property *scaling_mode_property;
	/**
	 * @aspect_ratio_property: Optional connector property to control the
	 * HDMI infoframe aspect ratio setting.
	 */
	struct drm_property *aspect_ratio_property;
	/**
	 * @content_type_property: Optional connector property to control the
	 * HDMI infoframe content type setting.
	 */
	struct drm_property *content_type_property;
	/**
	 * @degamma_lut_property: Optional CRTC property to set the LUT used to
	 * convert the framebuffer's colors to linear gamma.
	 */
	struct drm_property *degamma_lut_property;
	/**
	 * @degamma_lut_size_property: Optional CRTC property for the size of
	 * the degamma LUT as supported by the driver (read-only).
	 */
	struct drm_property *degamma_lut_size_property;
	/**
	 * @ctm_property: Optional CRTC property to set the
	 * matrix used to convert colors after the lookup in the
	 * degamma LUT.
	 */
	struct drm_property *ctm_property;
	/**
	 * @gamma_lut_property: Optional CRTC property to set the LUT used to
	 * convert the colors, after the CTM matrix, to the gamma space of the
	 * connected screen.
	 */
	struct drm_property *gamma_lut_property;
	/**
	 * @gamma_lut_size_property: Optional CRTC property for the size of the
	 * gamma LUT as supported by the driver (read-only).
	 */
	struct drm_property *gamma_lut_size_property;

	/**
	 * @suggested_x_property: Optional connector property with a hint for
	 * the position of the output on the host's screen.
	 */
	struct drm_property *suggested_x_property;
	/**
	 * @suggested_y_property: Optional connector property with a hint for
	 * the position of the output on the host's screen.
	 */
	struct drm_property *suggested_y_property;

	/**
	 * @non_desktop_property: Optional connector property with a hint
	 * that device isn't a standard display, and the console/desktop,
	 * should not be displayed on it.
	 */
	struct drm_property *non_desktop_property;

	/**
	 * @panel_orientation_property: Optional connector property indicating
	 * how the lcd-panel is mounted inside the casing (e.g. normal or
	 * upside-down).
	 */
	struct drm_property *panel_orientation_property;

	/**
	 * @writeback_fb_id_property: Property for writeback connectors, storing
	 * the ID of the output framebuffer.
	 * See also: drm_writeback_connector_init()
	 */
	struct drm_property *writeback_fb_id_property;

	/**
	 * @writeback_pixel_formats_property: Property for writeback connectors,
	 * storing an array of the supported pixel formats for the writeback
	 * engine (read-only).
	 * See also: drm_writeback_connector_init()
	 */
	struct drm_property *writeback_pixel_formats_property;
	/**
	 * @writeback_out_fence_ptr_property: Property for writeback connectors,
	 * fd pointer representing the outgoing fences for a writeback
	 * connector. Userspace should provide a pointer to a value of type s32,
	 * and then cast that pointer to u64.
	 * See also: drm_writeback_connector_init()
	 */
	struct drm_property *writeback_out_fence_ptr_property;

	/**
	 * @hdr_output_metadata_property: Connector property containing hdr
	 * metatada. This will be provided by userspace compositors based
	 * on HDR content
	 */
	struct drm_property *hdr_output_metadata_property;

	/**
	 * @content_protection_property: DRM ENUM property for content
	 * protection. See drm_connector_attach_content_protection_property().
	 */
	struct drm_property *content_protection_property;

	/**
	 * @hdcp_content_type_property: DRM ENUM property for type of
	 * Protected Content.
	 */
	struct drm_property *hdcp_content_type_property;

	/* dumb ioctl parameters */
	uint32_t preferred_depth, prefer_shadow;

	/**
	 * @quirk_addfb_prefer_xbgr_30bpp:
	 *
	 * Special hack for legacy ADDFB to keep nouveau userspace happy. Should
	 * only ever be set by the nouveau kernel driver.
	 */
	bool quirk_addfb_prefer_xbgr_30bpp;

	/**
	 * @quirk_addfb_prefer_host_byte_order:
	 *
	 * When set to true drm_mode_addfb() will pick host byte order
	 * pixel_format when calling drm_mode_addfb2().  This is how
	 * drm_mode_addfb() should have worked from day one.  It
	 * didn't though, so we ended up with quirks in both kernel
	 * and userspace drivers to deal with the broken behavior.
	 * Simply fixing drm_mode_addfb() unconditionally would break
	 * these drivers, so add a quirk bit here to allow drivers
	 * opt-in.
	 */
	bool quirk_addfb_prefer_host_byte_order;

	/**
	 * @async_page_flip: Does this device support async flips on the primary
	 * plane?
	 */
	bool async_page_flip;

	/**
	 * @fb_modifiers_not_supported:
	 *
	 * When this flag is set, the DRM device will not expose modifier
	 * support to userspace. This is only used by legacy drivers that infer
	 * the buffer layout through heuristics without using modifiers. New
	 * drivers shall not set fhis flag.
	 */
	bool fb_modifiers_not_supported;

	/**
	 * @normalize_zpos:
	 *
	 * If true the drm core will call drm_atomic_normalize_zpos() as part of
	 * atomic mode checking from drm_atomic_helper_check()
	 */
	bool normalize_zpos;

	/**
	 * @modifiers_property: Plane property to list support modifier/format
	 * combination.
	 */
	struct drm_property *modifiers_property;

	/* cursor size */
	/* 光标的最大宽度和高度 */
	uint32_t cursor_width, cursor_height;

	/**
	 * @suspend_state:
	 *
	 * Atomic state when suspended.
	 * Set by drm_mode_config_helper_suspend() and cleared by
	 * drm_mode_config_helper_resume().
	 */
	struct drm_atomic_state *suspend_state;

	const struct drm_mode_config_helper_funcs *helper_private;
};
```

### struct drm_mode_config_funcs

`drm_mode_config_funcs`用于管理显示设备显示模式的一组回调函数，为不同的显示模式配置任务提供必要的接口，每个 DRM 驱动程序可以根据硬件的特性，使用此结构体提供的回调函数来实现具体的功能。

```c
// include/drm/drm_mode_config.h

struct drm_mode_config_funcs {
	// 创建帧缓冲区，返回一个新的帧缓冲区对象
	struct drm_framebuffer *(*fb_create)(struct drm_device *dev,
						struct drm_file *file_priv,
						const struct drm_mode_fb_cmd2 *mode_cmd);
	// 返回帧缓冲区的显示格式信息
	const struct drm_format_info *(*get_format_info)(const struct 
						drm_mode_fb_cmd2 *mode_cmd);

	void (*output_poll_changed)(struct drm_device *dev);
	// 显示模式检查
	enum drm_mode_status (*mode_valid)(struct drm_device *dev,
					   const struct drm_display_mode *mode);

	int (*atomic_check)(struct drm_device *dev,
			    struct drm_atomic_state *state);
	// 更新显示模式
	int (*atomic_commit)(struct drm_device *dev,
			     struct drm_atomic_state *state,
			     bool nonblock);

	struct drm_atomic_state *(*atomic_state_alloc)(struct drm_device *dev);

	struct drm_atomic_state *(*atomic_state_alloc)(struct drm_device *dev);

	void (*atomic_state_free)(struct drm_atomic_state *state);
};
```

### 2.7 struct drm_minor

`struct drm_minor`用来描述在/dev下的drm设备节点，drm core会根据`driver_features`来决定是否为`drm_device`中的primary, render, accel注册字符设备，同时在`/dev/dri`目录下创建相应的设备节点。`drm_minor`用于管理DRM驱动中的此设备，将DRM设备划分为不同的设备节点，方便用户态程序访问和管理。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20241022164326.png)

```c
// include/drm/drm_file.h

/* Note that the values of this enum are ABI (it determines
 * /dev/dri/renderD* numbers).
 *
 * Setting DRM_MINOR_ACCEL to 32 gives enough space for more drm minors to
 * be implemented before we hit any future
 */
enum drm_minor_type {
	DRM_MINOR_PRIMARY = 0,
	DRM_MINOR_CONTROL = 1,
	DRM_MINOR_RENDER = 2,
	DRM_MINOR_ACCEL = 32,
};

/**
 * struct drm_minor - DRM device minor structure
 *
 * This structure represents a DRM minor number for device nodes in /dev.
 * Entirely opaque to drivers and should never be inspected directly by drivers.
 * Drivers instead should only interact with &struct drm_file and of course
 * &struct drm_device, which is also where driver-private data and resources can
 * be attached to.
 */
struct drm_minor {
	/* private: */
	/* 次设备号 */
	int index;			/* Minor device number */
	/* drm设备类型 */
	int type;                       /* Control or render or accel */
	struct device *kdev;		/* Linux device */
	/* 指向drm_drvice */
	struct drm_device *dev;
	/* debugfs 目录项 */
	struct dentry *debugfs_root;

	struct list_head debugfs_list;
	struct mutex debugfs_lock; /* Protects debugfs_list. */
};
```

### 2.8 struct drm_vblank_crtc

`struct drm_vblank_crtc`，和crtc相对应，用于管理crtc的垂直消隐相关状态的数据结构，在`drm_vblank_init()`中初始化

```c
// include/drm/drm_vblank.h
/**
 * struct drm_vblank_crtc - vblank tracking for a CRTC
 *
 * This structure tracks the vblank state for one CRTC.
 *
 * Note that for historical reasons - the vblank handling code is still shared
 * with legacy/non-kms drivers - this is a free-standing structure not directly
 * connected to &struct drm_crtc. But all public interface functions are taking
 * a &struct drm_crtc to hide this implementation detail.
 */
struct drm_vblank_crtc {
	/**
	 * @dev: Pointer to the &drm_device.
	 */
	// 指向drm_device
	struct drm_device *dev;
	/**
	 * @queue: Wait queue for vblank waiters.
	 */
	// 用于阻塞等待vblank事件
	wait_queue_head_t queue;
	/**
	 * @disable_timer: Disable timer for the delayed vblank disabling
	 * hysteresis logic. Vblank disabling is controlled through the
	 * drm_vblank_offdelay module option and the setting of the
	 * &drm_device.max_vblank_count value.
	 */
	struct timer_list disable_timer;

	/**
	 * @seqlock: Protect vblank count and time.
	 */
	seqlock_t seqlock;

	/**
	 * @count:
	 *
	 * Current software vblank counter.
	 *
	 * Note that for a given vblank counter value drm_crtc_handle_vblank()
	 * and drm_crtc_vblank_count() or drm_crtc_vblank_count_and_time()
	 * provide a barrier: Any writes done before calling
	 * drm_crtc_handle_vblank() will be visible to callers of the later
	 * functions, iff the vblank count is the same or a later one.
	 *
	 * IMPORTANT: This guarantee requires barriers, therefor never access
	 * this field directly. Use drm_crtc_vblank_count() instead.
	 */
	// vblank计数
	atomic64_t count;
	/**
	 * @time: Vblank timestamp corresponding to @count.
	 */
	// 最近一次vblank的时间戳
	ktime_t time;

	/**
	 * @refcount: Number of users/waiters of the vblank interrupt. Only when
	 * this refcount reaches 0 can the hardware interrupt be disabled using
	 * @disable_timer.
	 */
	// 引用计数 用于管理vblank资源
	atomic_t refcount;
	/**
	 * @last: Protected by &drm_device.vbl_lock, used for wraparound handling.
	 */
	u32 last;
	/**
	 * @max_vblank_count:
	 *
	 * Maximum value of the vblank registers for this crtc. This value +1
	 * will result in a wrap-around of the vblank register. It is used
	 * by the vblank core to handle wrap-arounds.
	 *
	 * If set to zero the vblank core will try to guess the elapsed vblanks
	 * between times when the vblank interrupt is disabled through
	 * high-precision timestamps. That approach is suffering from small
	 * races and imprecision over longer time periods, hence exposing a
	 * hardware vblank counter is always recommended.
	 *
	 * This is the runtime configurable per-crtc maximum set through
	 * drm_crtc_set_max_vblank_count(). If this is used the driver
	 * must leave the device wide &drm_device.max_vblank_count at zero.
	 *
	 * If non-zero, &drm_crtc_funcs.get_vblank_counter must be set.
	 */
	u32 max_vblank_count;
	/**
	 * @inmodeset: Tracks whether the vblank is disabled due to a modeset.
	 * For legacy driver bit 2 additionally tracks whether an additional
	 * temporary vblank reference has been acquired to paper over the
	 * hardware counter resetting/jumping. KMS drivers should instead just
	 * call drm_crtc_vblank_off() and drm_crtc_vblank_on(), which explicitly
	 * save and restore the vblank count.
	 */
	unsigned int inmodeset;
	/**
	 * @pipe: drm_crtc_index() of the &drm_crtc corresponding to this
	 * structure.
	 */
	// crtc的索引号
	unsigned int pipe;
	/**
	 * @framedur_ns: Frame/Field duration in ns, used by
	 * drm_crtc_vblank_helper_get_vblank_timestamp() and computed by
	 * drm_calc_timestamping_constants().
	 */
	int framedur_ns;
	/**
	 * @linedur_ns: Line duration in ns, used by
	 * drm_crtc_vblank_helper_get_vblank_timestamp() and computed by
	 * drm_calc_timestamping_constants().
	 */
	int linedur_ns;

	/**
	 * @hwmode:
	 *
	 * Cache of the current hardware display mode. Only valid when @enabled
	 * is set. This is used by helpers like
	 * drm_crtc_vblank_helper_get_vblank_timestamp(). We can't just access
	 * the hardware mode by e.g. looking at &drm_crtc_state.adjusted_mode,
	 * because that one is really hard to get from interrupt context.
	 */
	struct drm_display_mode hwmode;

	/**
	 * @enabled: Tracks the enabling state of the corresponding &drm_crtc to
	 * avoid double-disabling and hence corrupting saved state. Needed by
	 * drivers not using atomic KMS, since those might go through their CRTC
	 * disabling functions multiple times.
	 */
	bool enabled;

	/**
	 * @worker: The &kthread_worker used for executing vblank works.
	 */
	// vblank的工作线程
	struct kthread_worker *worker;

	/**
	 * @pending_work: A list of scheduled &drm_vblank_work items that are
	 * waiting for a future vblank.
	 */
	struct list_head pending_work;

	/**
	 * @work_wait_queue: The wait queue used for signaling that a
	 * &drm_vblank_work item has either finished executing, or was
	 * cancelled.
	 */
	wait_queue_head_t work_wait_queue;
};
```

### 2.9 struct drm_gem_object

linux内核使用`struct drm_gem_object`表示GEM对象

```c
// include/drm/drm_gem.h
/**
 * struct drm_gem_object - GEM buffer object
 *
 * This structure defines the generic parts for GEM buffer objects, which are
 * mostly around handling mmap and userspace handles.
 *
 * Buffer objects are often abbreviated to BO.
 */
struct drm_gem_object {
	/**
	 * @refcount:
	 *
	 * Reference count of this object
	 *
	 * Please use drm_gem_object_get() to acquire and drm_gem_object_put_locked()
	 * or drm_gem_object_put() to release a reference to a GEM
	 * buffer object.
	 */
	// 对象引用计数，用于对GEM对象生命周期管理
	struct kref refcount;

	/**
	 * @handle_count:
	 *
	 * This is the GEM file_priv handle count of this object.
	 *
	 * Each handle also holds a reference. Note that when the handle_count
	 * drops to 0 any global names (e.g. the id in the flink namespace) will
	 * be cleared.
	 *
	 * Protected by &drm_device.object_name_lock.
	 */
	unsigned handle_count;

	/**
	 * @dev: DRM dev this object belongs to.
	 */
	// 指向DRM设备
	struct drm_device *dev;

	/**
	 * @filp:
	 *
	 * SHMEM file node used as backing storage for swappable buffer objects.
	 * GEM also supports driver private objects with driver-specific backing
	 * storage (contiguous DMA memory, special reserved blocks). In this
	 * case @filp is NULL.
	 */
	struct file *filp;

	/**
	 * @vma_node:
	 *
	 * Mapping info for this object to support mmap. Drivers are supposed to
	 * allocate the mmap offset using drm_gem_create_mmap_offset(). The
	 * offset itself can be retrieved using drm_vma_node_offset_addr().
	 *
	 * Memory mapping itself is handled by drm_gem_mmap(), which also checks
	 * that userspace is allowed to access the object.
	 */
	struct drm_vma_offset_node vma_node;

	/**
	 * @size:
	 *
	 * Size of the object, in bytes.  Immutable over the object's
	 * lifetime.
	 */
	// 对象的大小，字节为单位
	size_t size;

	/**
	 * @name:
	 *
	 * Global name for this object, starts at 1. 0 means unnamed.
	 * Access is covered by &drm_device.object_name_lock. This is used by
	 * the GEM_FLINK and GEM_OPEN ioctls.
	 */
	// 对象
	int name;

	/**
	 * @dma_buf:
	 *
	 * dma-buf associated with this GEM object.
	 *
	 * Pointer to the dma-buf associated with this gem object (either
	 * through importing or exporting). We break the resulting reference
	 * loop when the last gem handle for this object is released.
	 *
	 * Protected by &drm_device.object_name_lock.
	 */
	struct dma_buf *dma_buf;

	/**
	 * @import_attach:
	 *
	 * dma-buf attachment backing this object.
	 *
	 * Any foreign dma_buf imported as a gem object has this set to the
	 * attachment point for the device. This is invariant over the lifetime
	 * of a gem object.
	 *
	 * The &drm_gem_object_funcs.free callback is responsible for
	 * cleaning up the dma_buf attachment and references acquired at import
	 * time.
	 *
	 * Note that the drm gem/prime core does not depend upon drivers setting
	 * this field any more. So for drivers where this doesn't make sense
	 * (e.g. virtual devices or a displaylink behind an usb bus) they can
	 * simply leave it as NULL.
	 */
	struct dma_buf_attachment *import_attach;

	/**
	 * @resv:
	 *
	 * Pointer to reservation object associated with the this GEM object.
	 *
	 * Normally (@resv == &@_resv) except for imported GEM objects.
	 */
	struct dma_resv *resv;

	/**
	 * @_resv:
	 *
	 * A reservation object for this GEM object.
	 *
	 * This is unused for imported GEM objects.
	 */
	struct dma_resv _resv;

	/**
	 * @gpuva:
	 *
	 * Provides the list of GPU VAs attached to this GEM object.
	 *
	 * Drivers should lock list accesses with the GEMs &dma_resv lock
	 * (&drm_gem_object.resv) or a custom lock if one is provided.
	 */
	struct {
		struct list_head list;

#ifdef CONFIG_LOCKDEP
		struct lockdep_map *lock_dep_map;
#endif
	} gpuva;

	/**
	 * @funcs:
	 *
	 * Optional GEM object functions. If this is set, it will be used instead of the
	 * corresponding &drm_driver GEM callbacks.
	 *
	 * New drivers should use this.
	 *
	 */
	const struct drm_gem_object_funcs *funcs;

	/**
	 * @lru_node:
	 *
	 * List node in a &drm_gem_lru.
	 */
	struct list_head lru_node;

	/**
	 * @lru:
	 *
	 * The current LRU list that the GEM object is on.
	 */
	struct drm_gem_lru *lru;
};


// struct drm_gem_object_funcs结构体
/**
 * struct drm_gem_object_funcs - GEM object functions
 */
struct drm_gem_object_funcs {
	/**
	 * @free:
	 *
	 * Deconstructor for drm_gem_objects.
	 *
	 * This callback is mandatory.
	 */
	void (*free)(struct drm_gem_object *obj);

	/**
	 * @open:
	 *
	 * Called upon GEM handle creation.
	 *
	 * This callback is optional.
	 */
	int (*open)(struct drm_gem_object *obj, struct drm_file *file);

	/**
	 * @close:
	 *
	 * Called upon GEM handle release.
	 *
	 * This callback is optional.
	 */
	void (*close)(struct drm_gem_object *obj, struct drm_file *file);

	/**
	 * @print_info:
	 *
	 * If driver subclasses struct &drm_gem_object, it can implement this
	 * optional hook for printing additional driver specific info.
	 *
	 * drm_printf_indent() should be used in the callback passing it the
	 * indent argument.
	 *
	 * This callback is called from drm_gem_print_info().
	 *
	 * This callback is optional.
	 */
	void (*print_info)(struct drm_printer *p, unsigned int indent,
			   const struct drm_gem_object *obj);

	/**
	 * @export:
	 *
	 * Export backing buffer as a &dma_buf.
	 * If this is not set drm_gem_prime_export() is used.
	 *
	 * This callback is optional.
	 */
	struct dma_buf *(*export)(struct drm_gem_object *obj, int flags);

	/**
	 * @pin:
	 *
	 * Pin backing buffer in memory. Used by the drm_gem_map_attach() helper.
	 *
	 * This callback is optional.
	 */
	int (*pin)(struct drm_gem_object *obj);

	/**
	 * @unpin:
	 *
	 * Unpin backing buffer. Used by the drm_gem_map_detach() helper.
	 *
	 * This callback is optional.
	 */
	void (*unpin)(struct drm_gem_object *obj);

	/**
	 * @get_sg_table:
	 *
	 * Returns a Scatter-Gather table representation of the buffer.
	 * Used when exporting a buffer by the drm_gem_map_dma_buf() helper.
	 * Releasing is done by calling dma_unmap_sg_attrs() and sg_free_table()
	 * in drm_gem_unmap_buf(), therefore these helpers and this callback
	 * here cannot be used for sg tables pointing at driver private memory
	 * ranges.
	 *
	 * See also drm_prime_pages_to_sg().
	 */
	struct sg_table *(*get_sg_table)(struct drm_gem_object *obj);

	/**
	 * @vmap:
	 *
	 * Returns a virtual address for the buffer. Used by the
	 * drm_gem_dmabuf_vmap() helper.
	 *
	 * This callback is optional.
	 */
	int (*vmap)(struct drm_gem_object *obj, struct iosys_map *map);

	/**
	 * @vunmap:
	 *
	 * Releases the address previously returned by @vmap. Used by the
	 * drm_gem_dmabuf_vunmap() helper.
	 *
	 * This callback is optional.
	 */
	void (*vunmap)(struct drm_gem_object *obj, struct iosys_map *map);

	/**
	 * @mmap:
	 *
	 * Handle mmap() of the gem object, setup vma accordingly.
	 *
	 * This callback is optional.
	 *
	 * The callback is used by both drm_gem_mmap_obj() and
	 * drm_gem_prime_mmap().  When @mmap is present @vm_ops is not
	 * used, the @mmap callback must set vma->vm_ops instead.
	 */
	int (*mmap)(struct drm_gem_object *obj, struct vm_area_struct *vma);

	/**
	 * @evict:
	 *
	 * Evicts gem object out from memory. Used by the drm_gem_object_evict()
	 * helper. Returns 0 on success, -errno otherwise.
	 *
	 * This callback is optional.
	 */
	int (*evict)(struct drm_gem_object *obj);

	/**
	 * @status:
	 *
	 * The optional status callback can return additional object state
	 * which determines which stats the object is counted against.  The
	 * callback is called under table_lock.  Racing against object status
	 * change is "harmless", and the callback can expect to not race
	 * against object destruction.
	 *
	 * Called by drm_show_memory_stats().
	 */
	enum drm_gem_object_status (*status)(struct drm_gem_object *obj);

	/**
	 * @vm_ops:
	 *
	 * Virtual memory operations used with mmap.
	 *
	 * This is optional but necessary for mmap support.
	 */
	const struct vm_operations_struct *vm_ops;
};
```

`struct drm_mm`，核心功能是对一段连续的地址空间进行区间分配和回收管理
```c
// include/drm/drm_mm.h

struct drm_vma_offset_manager {
	rwlock_t vm_lock;
	struct drm_mm vm_addr_space_mm;
};

/**
 * struct drm_mm - DRM allocator
 *
 * DRM range allocator with a few special functions and features geared towards
 * managing GPU memory. Except for the @color_adjust callback the structure is
 * entirely opaque and should only be accessed through the provided functions
 * and macros. This structure can be embedded into larger driver structures.
 */
struct drm_mm {
	/**
	 * @color_adjust:
	 *
	 * Optional driver callback to further apply restrictions on a hole. The
	 * node argument points at the node containing the hole from which the
	 * block would be allocated (see drm_mm_hole_follows() and friends). The
	 * other arguments are the size of the block to be allocated. The driver
	 * can adjust the start and end as needed to e.g. insert guard pages.
	 */
	void (*color_adjust)(const struct drm_mm_node *node,
			     unsigned long color,
			     u64 *start, u64 *end);

	/* private: */
	/* List of all memory nodes that immediately precede a free hole. */
	struct list_head hole_stack;
	/* head_node.node_list is the list of all memory nodes, ordered
	 * according to the (increasing) start address of the memory node. */
	struct drm_mm_node head_node;
	/* Keep an interval_tree for fast lookup of drm_mm_nodes by address. */
	struct rb_root_cached interval_tree;
	struct rb_root_cached holes_size;
	struct rb_root holes_addr;

	unsigned long scan_active;
};
```

`struct drm_mm_node`
```c
// include/drm/drm_mm.h

/**
 * struct drm_mm_node - allocated block in the DRM allocator
 *
 * This represents an allocated block in a &drm_mm allocator. Except for
 * pre-reserved nodes inserted using drm_mm_reserve_node() the structure is
 * entirely opaque and should only be accessed through the provided funcions.
 * Since allocation of these nodes is entirely handled by the driver they can be
 * embedded.
 */
struct drm_mm_node {
	/** @color: Opaque driver-private tag. */
	unsigned long color;
	/** @start: Start address of the allocated block. */
	u64 start;
	/** @size: Size of the allocated block. */
	u64 size;
	/* private: */
	struct drm_mm *mm;
	struct list_head node_list;
	struct list_head hole_stack;
	struct rb_node rb;
	struct rb_node rb_hole_size;
	struct rb_node rb_hole_addr;
	u64 __subtree_last;
	u64 hole_size;
	u64 subtree_max_hole;
	unsigned long flags;
#define DRM_MM_NODE_ALLOCATED_BIT	0
#define DRM_MM_NODE_SCANNED_BIT		1
#ifdef CONFIG_DRM_DEBUG_MM
	depot_stack_handle_t stack;
#endif
};
```

### 2.10 struct drm_driver

`struct drm_driver`用来描述drm驱动，drm显示驱动通常会静态初始化一个`drm_driver`结构体

```c
// include/drm/drm_drv.h
/**
 * struct drm_driver - DRM driver structure
 *
 * This structure represent the common code for a family of cards. There will be
 * one &struct drm_device for each card present in this family. It contains lots
 * of vfunc entries, and a pile of those probably should be moved to more
 * appropriate places like &drm_mode_config_funcs or into a new operations
 * structure for GEM drivers.
 */
struct drm_driver {
	/**
	 * @load:
	 *
	 * Backward-compatible driver callback to complete initialization steps
	 * after the driver is registered.  For this reason, may suffer from
	 * race conditions and its use is deprecated for new drivers.  It is
	 * therefore only supported for existing drivers not yet converted to
	 * the new scheme.  See devm_drm_dev_alloc() and drm_dev_register() for
	 * proper and race-free way to set up a &struct drm_device.
	 *
	 * This is deprecated, do not use!
	 *
	 * Returns:
	 *
	 * Zero on success, non-zero value on failure.
	 */
	int (*load) (struct drm_device *, unsigned long flags);

	/**
	 * @open:
	 *
	 * Driver callback when a new &struct drm_file is opened. Useful for
	 * setting up driver-private data structures like buffer allocators,
	 * execution contexts or similar things. Such driver-private resources
	 * must be released again in @postclose.
	 *
	 * Since the display/modeset side of DRM can only be owned by exactly
	 * one &struct drm_file (see &drm_file.is_master and &drm_device.master)
	 * there should never be a need to set up any modeset related resources
	 * in this callback. Doing so would be a driver design bug.
	 *
	 * Returns:
	 *
	 * 0 on success, a negative error code on failure, which will be
	 * promoted to userspace as the result of the open() system call.
	 */
	int (*open) (struct drm_device *, struct drm_file *);

	/**
	 * @postclose:
	 *
	 * One of the driver callbacks when a new &struct drm_file is closed.
	 * Useful for tearing down driver-private data structures allocated in
	 * @open like buffer allocators, execution contexts or similar things.
	 *
	 * Since the display/modeset side of DRM can only be owned by exactly
	 * one &struct drm_file (see &drm_file.is_master and &drm_device.master)
	 * there should never be a need to tear down any modeset related
	 * resources in this callback. Doing so would be a driver design bug.
	 */
	void (*postclose) (struct drm_device *, struct drm_file *);

	/**
	 * @lastclose:
	 *
	 * Called when the last &struct drm_file has been closed and there's
	 * currently no userspace client for the &struct drm_device.
	 *
	 * Modern drivers should only use this to force-restore the fbdev
	 * framebuffer using drm_fb_helper_restore_fbdev_mode_unlocked().
	 * Anything else would indicate there's something seriously wrong.
	 * Modern drivers can also use this to execute delayed power switching
	 * state changes, e.g. in conjunction with the :ref:`vga_switcheroo`
	 * infrastructure.
	 *
	 * This is called after @postclose hook has been called.
	 *
	 * NOTE:
	 *
	 * All legacy drivers use this callback to de-initialize the hardware.
	 * This is purely because of the shadow-attach model, where the DRM
	 * kernel driver does not really own the hardware. Instead ownershipe is
	 * handled with the help of userspace through an inheritedly racy dance
	 * to set/unset the VT into raw mode.
	 *
	 * Legacy drivers initialize the hardware in the @firstopen callback,
	 * which isn't even called for modern drivers.
	 */
	void (*lastclose) (struct drm_device *);

	/**
	 * @unload:
	 *
	 * Reverse the effects of the driver load callback.  Ideally,
	 * the clean up performed by the driver should happen in the
	 * reverse order of the initialization.  Similarly to the load
	 * hook, this handler is deprecated and its usage should be
	 * dropped in favor of an open-coded teardown function at the
	 * driver layer.  See drm_dev_unregister() and drm_dev_put()
	 * for the proper way to remove a &struct drm_device.
	 *
	 * The unload() hook is called right after unregistering
	 * the device.
	 *
	 */
	void (*unload) (struct drm_device *);

	/**
	 * @release:
	 *
	 * Optional callback for destroying device data after the final
	 * reference is released, i.e. the device is being destroyed.
	 *
	 * This is deprecated, clean up all memory allocations associated with a
	 * &drm_device using drmm_add_action(), drmm_kmalloc() and related
	 * managed resources functions.
	 */
	void (*release) (struct drm_device *);

	/**
	 * @master_set:
	 *
	 * Called whenever the minor master is set. Only used by vmwgfx.
	 */
	void (*master_set)(struct drm_device *dev, struct drm_file *file_priv,
			   bool from_open);
	/**
	 * @master_drop:
	 *
	 * Called whenever the minor master is dropped. Only used by vmwgfx.
	 */
	void (*master_drop)(struct drm_device *dev, struct drm_file *file_priv);

	/**
	 * @debugfs_init:
	 *
	 * Allows drivers to create driver-specific debugfs files.
	 */
	void (*debugfs_init)(struct drm_minor *minor);

	/**
	 * @gem_create_object: constructor for gem objects
	 *
	 * Hook for allocating the GEM object struct, for use by the CMA
	 * and SHMEM GEM helpers. Returns a GEM object on success, or an
	 * ERR_PTR()-encoded error code otherwise.
	 */
	struct drm_gem_object *(*gem_create_object)(struct drm_device *dev,
						    size_t size);

	/**
	 * @prime_handle_to_fd:
	 *
	 * PRIME export function. Only used by vmwgfx.
	 */
	int (*prime_handle_to_fd)(struct drm_device *dev, struct drm_file *file_priv,
				uint32_t handle, uint32_t flags, int *prime_fd);
	/**
	 * @prime_fd_to_handle:
	 *
	 * PRIME import function. Only used by vmwgfx.
	 */
	int (*prime_fd_to_handle)(struct drm_device *dev, struct drm_file *file_priv,
				int prime_fd, uint32_t *handle);

	/**
	 * @gem_prime_import:
	 *
	 * Import hook for GEM drivers.
	 *
	 * This defaults to drm_gem_prime_import() if not set.
	 */
	struct drm_gem_object * (*gem_prime_import)(struct drm_device *dev,
				struct dma_buf *dma_buf);
	/**
	 * @gem_prime_import_sg_table:
	 *
	 * Optional hook used by the PRIME helper functions
	 * drm_gem_prime_import() respectively drm_gem_prime_import_dev().
	 */
	struct drm_gem_object *(*gem_prime_import_sg_table)(
				struct drm_device *dev,
				struct dma_buf_attachment *attach,
				struct sg_table *sgt);

	/**
	 * @dumb_create:
	 *
	 * This creates a new dumb buffer in the driver's backing storage manager (GEM,
	 * TTM or something else entirely) and returns the resulting buffer handle. This
	 * handle can then be wrapped up into a framebuffer modeset object.
	 *
	 * Note that userspace is not allowed to use such objects for render
	 * acceleration - drivers must create their own private ioctls for such a use
	 * case.
	 *
	 * Width, height and depth are specified in the &drm_mode_create_dumb
	 * argument. The callback needs to fill the handle, pitch and size for
	 * the created buffer.
	 *
	 * Called by the user via ioctl.
	 *
	 * Returns:
	 *
	 * Zero on success, negative errno on failure.
	 */
	int (*dumb_create)(struct drm_file *file_priv,
			   struct drm_device *dev,
			   struct drm_mode_create_dumb *args);
	/**
	 * @dumb_map_offset:
	 *
	 * Allocate an offset in the drm device node's address space to be able to
	 * memory map a dumb buffer.
	 *
	 * The default implementation is drm_gem_create_mmap_offset(). GEM based
	 * drivers must not overwrite this.
	 *
	 * Called by the user via ioctl.
	 *
	 * Returns:
	 *
	 * Zero on success, negative errno on failure.
	 */
	int (*dumb_map_offset)(struct drm_file *file_priv,
			       struct drm_device *dev, uint32_t handle,
			       uint64_t *offset);

	/**
	 * @show_fdinfo:
	 *
	 * Print device specific fdinfo.  See Documentation/gpu/drm-usage-stats.rst.
	 */
	void (*show_fdinfo)(struct drm_printer *p, struct drm_file *f);

	/** @major: driver major number */
	/* 用于描述驱动版本 */
	int major;
	/** @minor: driver minor number */
	int minor;
	/** @patchlevel: driver patch level */
	int patchlevel;
	/** @name: driver name */
	/* 驱动名称 */
	char *name;
	/** @desc: driver description */
	char *desc;
	/** @date: driver date */
	char *date;

	/**
	 * @driver_features:
	 * Driver features, see &enum drm_driver_feature. Drivers can disable
	 * some features on a per-instance basis using
	 * &drm_device.driver_features.
	 */
	/* 驱动标志，每一位有特定的含义 */
	u32 driver_features;

	/**
	 * @ioctls:
	 *
	 * Array of driver-private IOCTL description entries. See the chapter on
	 * :ref:`IOCTL support in the userland interfaces
	 * chapter<drm_driver_ioctl>` for the full details.
	 */

	const struct drm_ioctl_desc *ioctls;
	/** @num_ioctls: Number of entries in @ioctls. */
	int num_ioctls;

	/**
	 * @fops:
	 *
	 * File operations for the DRM device node. See the discussion in
	 * :ref:`file operations<drm_driver_fops>` for in-depth coverage and
	 * some examples.
	 */
	const struct file_operations *fops;

#ifdef CONFIG_DRM_LEGACY
	/* Everything below here is for legacy driver, never use! */
	/* private: */

	int (*firstopen) (struct drm_device *);
	void (*preclose) (struct drm_device *, struct drm_file *file_priv);
	int (*dma_ioctl) (struct drm_device *dev, void *data, struct drm_file *file_priv);
	int (*dma_quiescent) (struct drm_device *);
	int (*context_dtor) (struct drm_device *dev, int context);
	irqreturn_t (*irq_handler)(int irq, void *arg);
	void (*irq_preinstall)(struct drm_device *dev);
	int (*irq_postinstall)(struct drm_device *dev);
	void (*irq_uninstall)(struct drm_device *dev);
	u32 (*get_vblank_counter)(struct drm_device *dev, unsigned int pipe);
	int (*enable_vblank)(struct drm_device *dev, unsigned int pipe);
	void (*disable_vblank)(struct drm_device *dev, unsigned int pipe);
	int dev_priv_size;
#endif
};
```

### struct drm_encoder

```c
// include/drm/drm_encoder.h
/**
 * struct drm_encoder - central DRM encoder structure
 * @dev: parent DRM device
 * @head: list management
 * @base: base KMS object
 * @name: human readable name, can be overwritten by the driver
 * @funcs: control functions, can be NULL for simple managed encoders
 * @helper_private: mid-layer private data
 *
 * CRTCs drive pixels to encoders, which convert them into signals
 * appropriate for a given connector or set of connectors.
 */
struct drm_encoder {
	// 对应的drm_device
	struct drm_device *dev;
	struct list_head head;

	struct drm_mode_object base;
	// encoder名称
	char *name;
	/**
	 * @encoder_type:
	 *
	 * One of the DRM_MODE_ENCODER_<foo> types in drm_mode.h. The following
	 * encoder types are defined thus far:
	 *
	 * - DRM_MODE_ENCODER_DAC for VGA and analog on DVI-I/DVI-A.
	 *
	 * - DRM_MODE_ENCODER_TMDS for DVI, HDMI and (embedded) DisplayPort.
	 *
	 * - DRM_MODE_ENCODER_LVDS for display panels, or in general any panel
	 *   with a proprietary parallel connector.
	 *
	 * - DRM_MODE_ENCODER_TVDAC for TV output (Composite, S-Video,
	 *   Component, SCART).
	 *
	 * - DRM_MODE_ENCODER_VIRTUAL for virtual machine displays
	 *
	 * - DRM_MODE_ENCODER_DSI for panels connected using the DSI serial bus.
	 *
	 * - DRM_MODE_ENCODER_DPI for panels connected using the DPI parallel
	 *   bus.
	 *
	 * - DRM_MODE_ENCODER_DPMST for special fake encoders used to allow
	 *   mutliple DP MST streams to share one physical encoder.
	 */
	// encoder类型
	int encoder_type;

	/**
	 * @index: Position inside the mode_config.list, can be used as an array
	 * index. It is invariant over the lifetime of the encoder.
	 */
	// 在drm_mode_config 链表中的索引
	unsigned index;

	/**
	 * @possible_crtcs: Bitmask of potential CRTC bindings, using
	 * drm_crtc_index() as the index into the bitfield. The driver must set
	 * the bits for all &drm_crtc objects this encoder can be connected to
	 * before calling drm_dev_register().
	 *
	 * You will get a WARN if you get this wrong in the driver.
	 *
	 * Note that since CRTC objects can't be hotplugged the assigned indices
	 * are stable and hence known before registering all objects.
	 */
	uint32_t possible_crtcs;

	/**
	 * @possible_clones: Bitmask of potential sibling encoders for cloning,
	 * using drm_encoder_index() as the index into the bitfield. The driver
	 * must set the bits for all &drm_encoder objects which can clone a
	 * &drm_crtc together with this encoder before calling
	 * drm_dev_register(). Drivers should set the bit representing the
	 * encoder itself, too. Cloning bits should be set such that when two
	 * encoders can be used in a cloned configuration, they both should have
	 * each another bits set.
	 *
	 * As an exception to the above rule if the driver doesn't implement
	 * any cloning it can leave @possible_clones set to 0. The core will
	 * automagically fix this up by setting the bit for the encoder itself.
	 *
	 * You will get a WARN if you get this wrong in the driver.
	 *
	 * Note that since encoder objects can't be hotplugged the assigned indices
	 * are stable and hence known before registering all objects.
	 */
	uint32_t possible_clones;

	/**
	 * @crtc: Currently bound CRTC, only really meaningful for non-atomic
	 * drivers.  Atomic drivers should instead check
	 * &drm_connector_state.crtc.
	 */
	// 当前绑定的crtc
	struct drm_crtc *crtc;

	/**
	 * @bridge_chain: Bridges attached to this encoder. Drivers shall not
	 * access this field directly.
	 */
	struct list_head bridge_chain;

	const struct drm_encoder_funcs *funcs;
	const struct drm_encoder_helper_funcs *helper_private;
};

// drm_encoder_funcs
/**
 * struct drm_encoder_funcs - encoder controls
 *
 * Encoders sit between CRTCs and connectors.
 */
struct drm_encoder_funcs {
	/**
	 * @reset:
	 *
	 * Reset encoder hardware and software state to off. This function isn't
	 * called by the core directly, only through drm_mode_config_reset().
	 * It's not a helper hook only for historical reasons.
	 */
	void (*reset)(struct drm_encoder *encoder);

	/**
	 * @destroy:
	 *
	 * Clean up encoder resources. This is only called at driver unload time
	 * through drm_mode_config_cleanup() since an encoder cannot be
	 * hotplugged in DRM.
	 */
	void (*destroy)(struct drm_encoder *encoder);

	/**
	 * @late_register:
	 *
	 * This optional hook can be used to register additional userspace
	 * interfaces attached to the encoder like debugfs interfaces.
	 * It is called late in the driver load sequence from drm_dev_register().
	 * Everything added from this callback should be unregistered in
	 * the early_unregister callback.
	 *
	 * Returns:
	 *
	 * 0 on success, or a negative error code on failure.
	 */
	int (*late_register)(struct drm_encoder *encoder);

	/**
	 * @early_unregister:
	 *
	 * This optional hook should be used to unregister the additional
	 * userspace interfaces attached to the encoder from
	 * @late_register. It is called from drm_dev_unregister(),
	 * early in the driver unload sequence to disable userspace access
	 * before data structures are torndown.
	 */
	void (*early_unregister)(struct drm_encoder *encoder);
};


// drm_encoder_helper_funcs
/**
 * struct drm_encoder_helper_funcs - helper operations for encoders
 *
 * These hooks are used by the legacy CRTC helpers and the new atomic
 * modesetting helpers.
 */
struct drm_encoder_helper_funcs {
	/**
	 * @dpms:
	 *
	 * Callback to control power levels on the encoder.  If the mode passed in
	 * is unsupported, the provider must use the next lowest power level.
	 * This is used by the legacy encoder helpers to implement DPMS
	 * functionality in drm_helper_connector_dpms().
	 *
	 * This callback is also used to disable an encoder by calling it with
	 * DRM_MODE_DPMS_OFF if the @disable hook isn't used.
	 *
	 * This callback is used by the legacy CRTC helpers.  Atomic helpers
	 * also support using this hook for enabling and disabling an encoder to
	 * facilitate transitions to atomic, but it is deprecated. Instead
	 * @enable and @disable should be used.
	 */
	void (*dpms)(struct drm_encoder *encoder, int mode);

	/**
	 * @mode_valid:
	 *
	 * This callback is used to check if a specific mode is valid in this
	 * encoder. This should be implemented if the encoder has some sort
	 * of restriction in the modes it can display. For example, a given
	 * encoder may be responsible to set a clock value. If the clock can
	 * not produce all the values for the available modes then this callback
	 * can be used to restrict the number of modes to only the ones that
	 * can be displayed.
	 *
	 * This hook is used by the probe helpers to filter the mode list in
	 * drm_helper_probe_single_connector_modes(), and it is used by the
	 * atomic helpers to validate modes supplied by userspace in
	 * drm_atomic_helper_check_modeset().
	 *
	 * This function is optional.
	 *
	 * NOTE:
	 *
	 * Since this function is both called from the check phase of an atomic
	 * commit, and the mode validation in the probe paths it is not allowed
	 * to look at anything else but the passed-in mode, and validate it
	 * against configuration-invariant hardward constraints. Any further
	 * limits which depend upon the configuration can only be checked in
	 * @mode_fixup or @atomic_check.
	 *
	 * RETURNS:
	 *
	 * drm_mode_status Enum
	 */
	enum drm_mode_status (*mode_valid)(struct drm_encoder *crtc,
					   const struct drm_display_mode *mode);

	/**
	 * @mode_fixup:
	 *
	 * This callback is used to validate and adjust a mode. The parameter
	 * mode is the display mode that should be fed to the next element in
	 * the display chain, either the final &drm_connector or a &drm_bridge.
	 * The parameter adjusted_mode is the input mode the encoder requires. It
	 * can be modified by this callback and does not need to match mode. See
	 * also &drm_crtc_state.adjusted_mode for more details.
	 *
	 * This function is used by both legacy CRTC helpers and atomic helpers.
	 * This hook is optional.
	 *
	 * NOTE:
	 *
	 * This function is called in the check phase of atomic modesets, which
	 * can be aborted for any reason (including on userspace's request to
	 * just check whether a configuration would be possible). Atomic drivers
	 * MUST NOT touch any persistent state (hardware or software) or data
	 * structures except the passed in adjusted_mode parameter.
	 *
	 * This is in contrast to the legacy CRTC helpers where this was
	 * allowed.
	 *
	 * Atomic drivers which need to inspect and adjust more state should
	 * instead use the @atomic_check callback. If @atomic_check is used,
	 * this hook isn't called since @atomic_check allows a strict superset
	 * of the functionality of @mode_fixup.
	 *
	 * Also beware that userspace can request its own custom modes, neither
	 * core nor helpers filter modes to the list of probe modes reported by
	 * the GETCONNECTOR IOCTL and stored in &drm_connector.modes. To ensure
	 * that modes are filtered consistently put any encoder constraints and
	 * limits checks into @mode_valid.
	 *
	 * RETURNS:
	 *
	 * True if an acceptable configuration is possible, false if the modeset
	 * operation should be rejected.
	 */
	bool (*mode_fixup)(struct drm_encoder *encoder,
			   const struct drm_display_mode *mode,
			   struct drm_display_mode *adjusted_mode);

	/**
	 * @prepare:
	 *
	 * This callback should prepare the encoder for a subsequent modeset,
	 * which in practice means the driver should disable the encoder if it
	 * is running. Most drivers ended up implementing this by calling their
	 * @dpms hook with DRM_MODE_DPMS_OFF.
	 *
	 * This callback is used by the legacy CRTC helpers.  Atomic helpers
	 * also support using this hook for disabling an encoder to facilitate
	 * transitions to atomic, but it is deprecated. Instead @disable should
	 * be used.
	 */
	void (*prepare)(struct drm_encoder *encoder);

	/**
	 * @commit:
	 *
	 * This callback should commit the new mode on the encoder after a modeset,
	 * which in practice means the driver should enable the encoder.  Most
	 * drivers ended up implementing this by calling their @dpms hook with
	 * DRM_MODE_DPMS_ON.
	 *
	 * This callback is used by the legacy CRTC helpers.  Atomic helpers
	 * also support using this hook for enabling an encoder to facilitate
	 * transitions to atomic, but it is deprecated. Instead @enable should
	 * be used.
	 */
	void (*commit)(struct drm_encoder *encoder);

	/**
	 * @mode_set:
	 *
	 * This callback is used to update the display mode of an encoder.
	 *
	 * Note that the display pipe is completely off when this function is
	 * called. Drivers which need hardware to be running before they program
	 * the new display mode (because they implement runtime PM) should not
	 * use this hook, because the helper library calls it only once and not
	 * every time the display pipeline is suspend using either DPMS or the
	 * new "ACTIVE" property. Such drivers should instead move all their
	 * encoder setup into the @enable callback.
	 *
	 * This callback is used both by the legacy CRTC helpers and the atomic
	 * modeset helpers. It is optional in the atomic helpers.
	 *
	 * NOTE:
	 *
	 * If the driver uses the atomic modeset helpers and needs to inspect
	 * the connector state or connector display info during mode setting,
	 * @atomic_mode_set can be used instead.
	 */
	void (*mode_set)(struct drm_encoder *encoder,
			 struct drm_display_mode *mode,
			 struct drm_display_mode *adjusted_mode);

	/**
	 * @atomic_mode_set:
	 *
	 * This callback is used to update the display mode of an encoder.
	 *
	 * Note that the display pipe is completely off when this function is
	 * called. Drivers which need hardware to be running before they program
	 * the new display mode (because they implement runtime PM) should not
	 * use this hook, because the helper library calls it only once and not
	 * every time the display pipeline is suspended using either DPMS or the
	 * new "ACTIVE" property. Such drivers should instead move all their
	 * encoder setup into the @enable callback.
	 *
	 * This callback is used by the atomic modeset helpers in place of the
	 * @mode_set callback, if set by the driver. It is optional and should
	 * be used instead of @mode_set if the driver needs to inspect the
	 * connector state or display info, since there is no direct way to
	 * go from the encoder to the current connector.
	 */
	void (*atomic_mode_set)(struct drm_encoder *encoder,
				struct drm_crtc_state *crtc_state,
				struct drm_connector_state *conn_state);

	/**
	 * @detect:
	 *
	 * This callback can be used by drivers who want to do detection on the
	 * encoder object instead of in connector functions.
	 *
	 * It is not used by any helper and therefore has purely driver-specific
	 * semantics. New drivers shouldn't use this and instead just implement
	 * their own private callbacks.
	 *
	 * FIXME:
	 *
	 * This should just be converted into a pile of driver vfuncs.
	 * Currently radeon, amdgpu and nouveau are using it.
	 */
	enum drm_connector_status (*detect)(struct drm_encoder *encoder,
					    struct drm_connector *connector);

	/**
	 * @atomic_disable:
	 *
	 * This callback should be used to disable the encoder. With the atomic
	 * drivers it is called before this encoder's CRTC has been shut off
	 * using their own &drm_crtc_helper_funcs.atomic_disable hook. If that
	 * sequence is too simple drivers can just add their own driver private
	 * encoder hooks and call them from CRTC's callback by looping over all
	 * encoders connected to it using for_each_encoder_on_crtc().
	 *
	 * This callback is a variant of @disable that provides the atomic state
	 * to the driver. If @atomic_disable is implemented, @disable is not
	 * called by the helpers.
	 *
	 * This hook is only used by atomic helpers. Atomic drivers don't need
	 * to implement it if there's no need to disable anything at the encoder
	 * level. To ensure that runtime PM handling (using either DPMS or the
	 * new "ACTIVE" property) works @atomic_disable must be the inverse of
	 * @atomic_enable.
	 */
	void (*atomic_disable)(struct drm_encoder *encoder,
			       struct drm_atomic_state *state);

	/**
	 * @atomic_enable:
	 *
	 * This callback should be used to enable the encoder. It is called
	 * after this encoder's CRTC has been enabled using their own
	 * &drm_crtc_helper_funcs.atomic_enable hook. If that sequence is
	 * too simple drivers can just add their own driver private encoder
	 * hooks and call them from CRTC's callback by looping over all encoders
	 * connected to it using for_each_encoder_on_crtc().
	 *
	 * This callback is a variant of @enable that provides the atomic state
	 * to the driver. If @atomic_enable is implemented, @enable is not
	 * called by the helpers.
	 *
	 * This hook is only used by atomic helpers, it is the opposite of
	 * @atomic_disable. Atomic drivers don't need to implement it if there's
	 * no need to enable anything at the encoder level. To ensure that
	 * runtime PM handling works @atomic_enable must be the inverse of
	 * @atomic_disable.
	 */
	void (*atomic_enable)(struct drm_encoder *encoder,
			      struct drm_atomic_state *state);

	/**
	 * @disable:
	 *
	 * This callback should be used to disable the encoder. With the atomic
	 * drivers it is called before this encoder's CRTC has been shut off
	 * using their own &drm_crtc_helper_funcs.disable hook.  If that
	 * sequence is too simple drivers can just add their own driver private
	 * encoder hooks and call them from CRTC's callback by looping over all
	 * encoders connected to it using for_each_encoder_on_crtc().
	 *
	 * This hook is used both by legacy CRTC helpers and atomic helpers.
	 * Atomic drivers don't need to implement it if there's no need to
	 * disable anything at the encoder level. To ensure that runtime PM
	 * handling (using either DPMS or the new "ACTIVE" property) works
	 * @disable must be the inverse of @enable for atomic drivers.
	 *
	 * For atomic drivers also consider @atomic_disable and save yourself
	 * from having to read the NOTE below!
	 *
	 * NOTE:
	 *
	 * With legacy CRTC helpers there's a big semantic difference between
	 * @disable and other hooks (like @prepare or @dpms) used to shut down a
	 * encoder: @disable is only called when also logically disabling the
	 * display pipeline and needs to release any resources acquired in
	 * @mode_set (like shared PLLs, or again release pinned framebuffers).
	 *
	 * Therefore @disable must be the inverse of @mode_set plus @commit for
	 * drivers still using legacy CRTC helpers, which is different from the
	 * rules under atomic.
	 */
	void (*disable)(struct drm_encoder *encoder);

	/**
	 * @enable:
	 *
	 * This callback should be used to enable the encoder. With the atomic
	 * drivers it is called after this encoder's CRTC has been enabled using
	 * their own &drm_crtc_helper_funcs.enable hook.  If that sequence is
	 * too simple drivers can just add their own driver private encoder
	 * hooks and call them from CRTC's callback by looping over all encoders
	 * connected to it using for_each_encoder_on_crtc().
	 *
	 * This hook is only used by atomic helpers, it is the opposite of
	 * @disable. Atomic drivers don't need to implement it if there's no
	 * need to enable anything at the encoder level. To ensure that
	 * runtime PM handling (using either DPMS or the new "ACTIVE" property)
	 * works @enable must be the inverse of @disable for atomic drivers.
	 */
	void (*enable)(struct drm_encoder *encoder);

	/**
	 * @atomic_check:
	 *
	 * This callback is used to validate encoder state for atomic drivers.
	 * Since the encoder is the object connecting the CRTC and connector it
	 * gets passed both states, to be able to validate interactions and
	 * update the CRTC to match what the encoder needs for the requested
	 * connector.
	 *
	 * Since this provides a strict superset of the functionality of
	 * @mode_fixup (the requested and adjusted modes are both available
	 * through the passed in &struct drm_crtc_state) @mode_fixup is not
	 * called when @atomic_check is implemented.
	 *
	 * This function is used by the atomic helpers, but it is optional.
	 *
	 * NOTE:
	 *
	 * This function is called in the check phase of an atomic update. The
	 * driver is not allowed to change anything outside of the free-standing
	 * state objects passed-in or assembled in the overall &drm_atomic_state
	 * update tracking structure.
	 *
	 * Also beware that userspace can request its own custom modes, neither
	 * core nor helpers filter modes to the list of probe modes reported by
	 * the GETCONNECTOR IOCTL and stored in &drm_connector.modes. To ensure
	 * that modes are filtered consistently put any encoder constraints and
	 * limits checks into @mode_valid.
	 *
	 * RETURNS:
	 *
	 * 0 on success, -EINVAL if the state or the transition can't be
	 * supported, -ENOMEM on memory allocation failure and -EDEADLK if an
	 * attempt to obtain another state object ran into a &drm_modeset_lock
	 * deadlock.
	 */
	int (*atomic_check)(struct drm_encoder *encoder,
			    struct drm_crtc_state *crtc_state,
			    struct drm_connector_state *conn_state);
};
```

### struct drm_connector

```c
// include/drm/drm_connector.h
/**
 * struct drm_connector - central DRM connector control structure
 *
 * Each connector may be connected to one or more CRTCs, or may be clonable by
 * another connector if they can share a CRTC.  Each connector also has a specific
 * position in the broader display (referred to as a 'screen' though it could
 * span multiple monitors).
 */
struct drm_connector {
	// 对应的drm_device
	/** @dev: parent DRM device */
	struct drm_device *dev;
	/** @kdev: kernel device for sysfs attributes */
	struct device *kdev;
	/** @attr: sysfs attributes */
	struct device_attribute *attr;
	/**
	 * @fwnode: associated fwnode supplied by platform firmware
	 *
	 * Drivers can set this to associate a fwnode with a connector, drivers
	 * are expected to get a reference on the fwnode when setting this.
	 * drm_connector_cleanup() will call fwnode_handle_put() on this.
	 */
	struct fwnode_handle *fwnode;

	/**
	 * @head:
	 *
	 * List of all connectors on a @dev, linked from
	 * &drm_mode_config.connector_list. Protected by
	 * &drm_mode_config.connector_list_lock, but please only use
	 * &drm_connector_list_iter to walk this list.
	 */
	// 用于链接到drm_mode_config上的connector_list链表
	struct list_head head;

	/**
	 * @global_connector_list_entry:
	 *
	 * Connector entry in the global connector-list, used by
	 * drm_connector_find_by_fwnode().
	 */
	struct list_head global_connector_list_entry;

	/** @base: base KMS object */
	// 基类
	struct drm_mode_object base;

	/** @name: human readable name, can be overwritten by the driver */
	// connector的名称
	char *name;

	/**
	 * @mutex: Lock for general connector state, but currently only protects
	 * @registered. Most of the connector state is still protected by
	 * &drm_mode_config.mutex.
	 */
	struct mutex mutex;

	/**
	 * @index: Compacted connector index, which matches the position inside
	 * the mode_config.list for drivers not supporting hot-add/removing. Can
	 * be used as an array index. It is invariant over the lifetime of the
	 * connector.
	 */
	unsigned index;

	/**
	 * @connector_type:
	 * one of the DRM_MODE_CONNECTOR_<foo> types from drm_mode.h
	 */
	// connector的类型
	int connector_type;
	/** @connector_type_id: index into connector type enum */
	int connector_type_id;
	/**
	 * @interlace_allowed:
	 * Can this connector handle interlaced modes? Only used by
	 * drm_helper_probe_single_connector_modes() for mode filtering.
	 */
	bool interlace_allowed;
	/**
	 * @doublescan_allowed:
	 * Can this connector handle doublescan? Only used by
	 * drm_helper_probe_single_connector_modes() for mode filtering.
	 */
	bool doublescan_allowed;
	/**
	 * @stereo_allowed:
	 * Can this connector handle stereo modes? Only used by
	 * drm_helper_probe_single_connector_modes() for mode filtering.
	 */
	bool stereo_allowed;

	/**
	 * @ycbcr_420_allowed : This bool indicates if this connector is
	 * capable of handling YCBCR 420 output. While parsing the EDID
	 * blocks it's very helpful to know if the source is capable of
	 * handling YCBCR 420 outputs.
	 */
	bool ycbcr_420_allowed;

	/**
	 * @registration_state: Is this connector initializing, exposed
	 * (registered) with userspace, or unregistered?
	 *
	 * Protected by @mutex.
	 */
	// 当前的注册状态
	enum drm_connector_registration_state registration_state;

	/**
	 * @modes:
	 * Modes available on this connector (from fill_modes() + user).
	 * Protected by &drm_mode_config.mutex.
	 */
	struct list_head modes;

	/**
	 * @status:
	 * One of the drm_connector_status enums (connected, not, or unknown).
	 * Protected by &drm_mode_config.mutex.
	 */
	// 这里表示connector的连接状态
	enum drm_connector_status status;

	/**
	 * @probed_modes:
	 * These are modes added by probing with DDC or the BIOS, before
	 * filtering is applied. Used by the probe helpers. Protected by
	 * &drm_mode_config.mutex.
	 */
	struct list_head probed_modes;

	/**
	 * @display_info: Display information is filled from EDID information
	 * when a display is detected. For non hot-pluggable displays such as
	 * flat panels in embedded systems, the driver should initialize the
	 * &drm_display_info.width_mm and &drm_display_info.height_mm fields
	 * with the physical size of the display.
	 *
	 * Protected by &drm_mode_config.mutex.
	 */
	struct drm_display_info display_info;

	/** @funcs: connector control functions */
	const struct drm_connector_funcs *funcs;

	/**
	 * @edid_blob_ptr: DRM property containing EDID if present. Protected by
	 * &drm_mode_config.mutex. This should be updated only by calling
	 * drm_connector_update_edid_property().
	 */
	struct drm_property_blob *edid_blob_ptr;

	/** @properties: property tracking for this connector */
	struct drm_object_properties properties;

	/**
	 * @scaling_mode_property: Optional atomic property to control the
	 * upscaling. See drm_connector_attach_content_protection_property().
	 */
	struct drm_property *scaling_mode_property;

	/**
	 * @vrr_capable_property: Optional property to help userspace
	 * query hardware support for variable refresh rate on a connector.
	 * connector. Drivers can add the property to a connector by
	 * calling drm_connector_attach_vrr_capable_property().
	 *
	 * This should be updated only by calling
	 * drm_connector_set_vrr_capable_property().
	 */
	struct drm_property *vrr_capable_property;

	/**
	 * @colorspace_property: Connector property to set the suitable
	 * colorspace supported by the sink.
	 */
	struct drm_property *colorspace_property;

	/**
	 * @path_blob_ptr:
	 *
	 * DRM blob property data for the DP MST path property. This should only
	 * be updated by calling drm_connector_set_path_property().
	 */
	struct drm_property_blob *path_blob_ptr;

	/**
	 * @max_bpc_property: Default connector property for the max bpc to be
	 * driven out of the connector.
	 */
	struct drm_property *max_bpc_property;

	/** @privacy_screen: drm_privacy_screen for this connector, or NULL. */
	struct drm_privacy_screen *privacy_screen;

	/** @privacy_screen_notifier: privacy-screen notifier_block */
	struct notifier_block privacy_screen_notifier;

	/**
	 * @privacy_screen_sw_state_property: Optional atomic property for the
	 * connector to control the integrated privacy screen.
	 */
	struct drm_property *privacy_screen_sw_state_property;

	/**
	 * @privacy_screen_hw_state_property: Optional atomic property for the
	 * connector to report the actual integrated privacy screen state.
	 */
	struct drm_property *privacy_screen_hw_state_property;

#define DRM_CONNECTOR_POLL_HPD (1 << 0)
#define DRM_CONNECTOR_POLL_CONNECT (1 << 1)
#define DRM_CONNECTOR_POLL_DISCONNECT (1 << 2)

	/**
	 * @polled:
	 *
	 * Connector polling mode, a combination of
	 *
	 * DRM_CONNECTOR_POLL_HPD
	 *     The connector generates hotplug events and doesn't need to be
	 *     periodically polled. The CONNECT and DISCONNECT flags must not
	 *     be set together with the HPD flag.
	 *
	 * DRM_CONNECTOR_POLL_CONNECT
	 *     Periodically poll the connector for connection.
	 *
	 * DRM_CONNECTOR_POLL_DISCONNECT
	 *     Periodically poll the connector for disconnection, without
	 *     causing flickering even when the connector is in use. DACs should
	 *     rarely do this without a lot of testing.
	 *
	 * Set to 0 for connectors that don't support connection status
	 * discovery.
	 */
	// 轮询检测显示接口热插拔
	uint8_t polled;

	/**
	 * @dpms: Current dpms state. For legacy drivers the
	 * &drm_connector_funcs.dpms callback must update this. For atomic
	 * drivers, this is handled by the core atomic code, and drivers must
	 * only take &drm_crtc_state.active into account.
	 */
	int dpms;

	/** @helper_private: mid-layer private data */
	const struct drm_connector_helper_funcs *helper_private;

	/** @cmdline_mode: mode line parsed from the kernel cmdline for this connector */
	struct drm_cmdline_mode cmdline_mode;
	/** @force: a DRM_FORCE_<foo> state for forced mode sets */
	enum drm_connector_force force;

	/**
	 * @edid_override: Override EDID set via debugfs.
	 *
	 * Do not modify or access outside of the drm_edid_override_* family of
	 * functions.
	 */
	const struct drm_edid *edid_override;

	/**
	 * @edid_override_mutex: Protect access to edid_override.
	 */
	struct mutex edid_override_mutex;

	/** @epoch_counter: used to detect any other changes in connector, besides status */
	u64 epoch_counter;

	/**
	 * @possible_encoders: Bit mask of encoders that can drive this
	 * connector, drm_encoder_index() determines the index into the bitfield
	 * and the bits are set with drm_connector_attach_encoder().
	 */
	// 支持的encoder
	u32 possible_encoders;

	/**
	 * @encoder: Currently bound encoder driving this connector, if any.
	 * Only really meaningful for non-atomic drivers. Atomic drivers should
	 * instead look at &drm_connector_state.best_encoder, and in case they
	 * need the CRTC driving this output, &drm_connector_state.crtc.
	 */
	// 当前绑定的encoder
	struct drm_encoder *encoder;

#define MAX_ELD_BYTES	128
	/** @eld: EDID-like data, if present */
	uint8_t eld[MAX_ELD_BYTES];
	/** @latency_present: AV delay info from ELD, if found */
	bool latency_present[2];
	/**
	 * @video_latency: Video latency info from ELD, if found.
	 * [0]: progressive, [1]: interlaced
	 */
	int video_latency[2];
	/**
	 * @audio_latency: audio latency info from ELD, if found
	 * [0]: progressive, [1]: interlaced
	 */
	int audio_latency[2];

	/**
	 * @ddc: associated ddc adapter.
	 * A connector usually has its associated ddc adapter. If a driver uses
	 * this field, then an appropriate symbolic link is created in connector
	 * sysfs directory to make it easy for the user to tell which i2c
	 * adapter is for a particular display.
	 *
	 * The field should be set by calling drm_connector_init_with_ddc().
	 */
	struct i2c_adapter *ddc;

	/**
	 * @null_edid_counter: track sinks that give us all zeros for the EDID.
	 * Needed to workaround some HW bugs where we get all 0s
	 */
	int null_edid_counter;

	/** @bad_edid_counter: track sinks that give us an EDID with invalid checksum */
	unsigned bad_edid_counter;

	/**
	 * @edid_corrupt: Indicates whether the last read EDID was corrupt. Used
	 * in Displayport compliance testing - Displayport Link CTS Core 1.2
	 * rev1.1 4.2.2.6
	 */
	bool edid_corrupt;
	/**
	 * @real_edid_checksum: real edid checksum for corrupted edid block.
	 * Required in Displayport 1.4 compliance testing
	 * rev1.1 4.2.2.6
	 */
	u8 real_edid_checksum;

	/** @debugfs_entry: debugfs directory for this connector */
	struct dentry *debugfs_entry;

	/**
	 * @state:
	 *
	 * Current atomic state for this connector.
	 *
	 * This is protected by &drm_mode_config.connection_mutex. Note that
	 * nonblocking atomic commits access the current connector state without
	 * taking locks. Either by going through the &struct drm_atomic_state
	 * pointers, see for_each_oldnew_connector_in_state(),
	 * for_each_old_connector_in_state() and
	 * for_each_new_connector_in_state(). Or through careful ordering of
	 * atomic commit operations as implemented in the atomic helpers, see
	 * &struct drm_crtc_commit.
	 */
	struct drm_connector_state *state;

	/* DisplayID bits. FIXME: Extract into a substruct? */

	/**
	 * @tile_blob_ptr:
	 *
	 * DRM blob property data for the tile property (used mostly by DP MST).
	 * This is meant for screens which are driven through separate display
	 * pipelines represented by &drm_crtc, which might not be running with
	 * genlocked clocks. For tiled panels which are genlocked, like
	 * dual-link LVDS or dual-link DSI, the driver should try to not expose
	 * the tiling and virtualize both &drm_crtc and &drm_plane if needed.
	 *
	 * This should only be updated by calling
	 * drm_connector_set_tile_property().
	 */
	struct drm_property_blob *tile_blob_ptr;

	/** @has_tile: is this connector connected to a tiled monitor */
	// 表示当前显示器是由多个平铺的小显示器组成，该connector仅仅是其中的一小块显示器
	bool has_tile;
	/** @tile_group: tile group for the connected monitor */
	struct drm_tile_group *tile_group;
	/** @tile_is_single_monitor: whether the tile is one monitor housing */
	bool tile_is_single_monitor;

	/** @num_h_tile: number of horizontal tiles in the tile group */
	/** @num_v_tile: number of vertical tiles in the tile group */
	uint8_t num_h_tile, num_v_tile;
	/** @tile_h_loc: horizontal location of this tile */
	/** @tile_v_loc: vertical location of this tile */
	uint8_t tile_h_loc, tile_v_loc;
	/** @tile_h_size: horizontal size of this tile. */
	/** @tile_v_size: vertical size of this tile. */
	uint16_t tile_h_size, tile_v_size;

	/**
	 * @free_node:
	 *
	 * List used only by &drm_connector_list_iter to be able to clean up a
	 * connector from any context, in conjunction with
	 * &drm_mode_config.connector_free_work.
	 */
	struct llist_node free_node;

	/** @hdr_sink_metadata: HDR Metadata Information read from sink */
	struct hdr_sink_metadata hdr_sink_metadata;
};
```

### struct drm_fb_helper

`struct drm_fb_helper`用于支持framebuffer兼容，主要帮助传统的帧缓冲区应用程序和现代的drm驱动程序进行交互，允许这些应用程序继续工作，同时让drm管理显示资源。

```c
/**
 * struct drm_fb_helper - main structure to emulate fbdev on top of KMS
 * @fb: Scanout framebuffer object
 * @dev: DRM device
 * @funcs: driver callbacks for fb helper
 * @info: emulated fbdev device info struct
 * @pseudo_palette: fake palette of 16 colors
 * @damage_clip: clip rectangle used with deferred_io to accumulate damage to
 *                the screen buffer
 * @damage_lock: spinlock protecting @damage_clip
 * @damage_work: worker used to flush the framebuffer
 * @resume_work: worker used during resume if the console lock is already taken
 *
 * This is the main structure used by the fbdev helpers. Drivers supporting
 * fbdev emulation should embedded this into their overall driver structure.
 * Drivers must also fill out a &struct drm_fb_helper_funcs with a few
 * operations.
 */
struct drm_fb_helper {
	/**
	 * @client:
	 *
	 * DRM client used by the generic fbdev emulation.
	 */
	struct drm_client_dev client;

	/**
	 * @buffer:
	 *
	 * Framebuffer used by the generic fbdev emulation.
	 */
	struct drm_client_buffer *buffer;
	// 指向drm_framebuffer
	struct drm_framebuffer *fb;
	struct drm_device *dev;
	const struct drm_fb_helper_funcs *funcs;
	struct fb_info *info;
	u32 pseudo_palette[17];
	struct drm_clip_rect damage_clip;
	spinlock_t damage_lock;
	struct work_struct damage_work;
	struct work_struct resume_work;

	/**
	 * @lock:
	 *
	 * Top-level FBDEV helper lock. This protects all internal data
	 * structures and lists, such as @connector_info and @crtc_info.
	 *
	 * FIXME: fbdev emulation locking is a mess and long term we want to
	 * protect all helper internal state with this lock as well as reduce
	 * core KMS locking as much as possible.
	 */
	struct mutex lock;

	/**
	 * @kernel_fb_list:
	 *
	 * Entry on the global kernel_fb_helper_list, used for kgdb entry/exit.
	 */
	struct list_head kernel_fb_list;

	/**
	 * @delayed_hotplug:
	 *
	 * A hotplug was received while fbdev wasn't in control of the DRM
	 * device, i.e. another KMS master was active. The output configuration
	 * needs to be reprobe when fbdev is in control again.
	 */
	bool delayed_hotplug;

	/**
	 * @deferred_setup:
	 *
	 * If no outputs are connected (disconnected or unknown) the FB helper
	 * code will defer setup until at least one of the outputs shows up.
	 * This field keeps track of the status so that setup can be retried
	 * at every hotplug event until it succeeds eventually.
	 *
	 * Protected by @lock.
	 */
	bool deferred_setup;

	/**
	 * @preferred_bpp:
	 *
	 * Temporary storage for the driver's preferred BPP setting passed to
	 * FB helper initialization. This needs to be tracked so that deferred
	 * FB helper setup can pass this on.
	 *
	 * See also: @deferred_setup
	 */
	int preferred_bpp;

#ifdef CONFIG_FB_DEFERRED_IO
	/**
	 * @fbdefio:
	 *
	 * Temporary storage for the driver's FB deferred I/O handler. If the
	 * driver uses the DRM fbdev emulation layer, this is set by the core
	 * to a generic deferred I/O handler if a driver is preferring to use
	 * a shadow buffer.
	 */
	struct fb_deferred_io fbdefio;
#endif
};
```

### struct drm_client_dev

`drm_client_dev`用于表示drm客户端设备，它在用于空间应用程序和drm驱动之间起到了桥梁作用，帮助管理客户端设备的显示资源和操作，用于简化drm的操作。`drm_client_dev`和`drm_device`是一个层级的，`drm_client_dev`位于`drm_device->drm_fb_helper->client`

```c
/**
 * struct drm_client_dev - DRM client instance
 */
struct drm_client_dev {
	/**
	 * @dev: DRM device
	 */
	struct drm_device *dev;

	/**
	 * @name: Name of the client.
	 */
	const char *name;

	/**
	 * @list:
	 *
	 * List of all clients of a DRM device, linked into
	 * &drm_device.clientlist. Protected by &drm_device.clientlist_mutex.
	 */
	struct list_head list;

	/**
	 * @funcs: DRM client functions (optional)
	 */
	const struct drm_client_funcs *funcs;

	/**
	 * @file: DRM file
	 */
	struct drm_file *file;

	/**
	 * @modeset_mutex: Protects @modesets.
	 */
	struct mutex modeset_mutex;

	/**
	 * @modesets: CRTC configurations
	 */
	struct drm_mode_set *modesets;

	/**
	 * @hotplug_failed:
	 *
	 * Set by client hotplug helpers if the hotplugging failed
	 * before. It is usually not tried again.
	 */
	bool hotplug_failed;
};

```

## 3. 显存管理

### 3.1 drm驱动内存资源管理

drm使用`drm_mm`来管理drm驱动分配的内存，它使用一个`drm_mm_node`链表代表所有被占用的内存区域

```c
// include/drm/drm_mm.h
/**
 * struct drm_mm - DRM allocator
 *
 * DRM range allocator with a few special functions and features geared towards
 * managing GPU memory. Except for the @color_adjust callback the structure is
 * entirely opaque and should only be accessed through the provided functions
 * and macros. This structure can be embedded into larger driver structures.
 */
struct drm_mm {
	/**
	 * @color_adjust:
	 *
	 * Optional driver callback to further apply restrictions on a hole. The
	 * node argument points at the node containing the hole from which the
	 * block would be allocated (see drm_mm_hole_follows() and friends). The
	 * other arguments are the size of the block to be allocated. The driver
	 * can adjust the start and end as needed to e.g. insert guard pages.
	 */
	void (*color_adjust)(const struct drm_mm_node *node,
			     unsigned long color,
			     u64 *start, u64 *end);

	/* private: */
	/* List of all memory nodes that immediately precede a free hole. */
	struct list_head hole_stack;
	/* head_node.node_list is the list of all memory nodes, ordered
	 * according to the (increasing) start address of the memory node. */
	struct drm_mm_node head_node;
	/* Keep an interval_tree for fast lookup of drm_mm_nodes by address. */
	struct rb_root_cached interval_tree;
	struct rb_root_cached holes_size;
	struct rb_root holes_addr;

	unsigned long scan_active;
};

/**
 * struct drm_mm_node - allocated block in the DRM allocator
 *
 * This represents an allocated block in a &drm_mm allocator. Except for
 * pre-reserved nodes inserted using drm_mm_reserve_node() the structure is
 * entirely opaque and should only be accessed through the provided funcions.
 * Since allocation of these nodes is entirely handled by the driver they can be
 * embedded.
 */
struct drm_mm_node {
	/** @color: Opaque driver-private tag. */
	unsigned long color;
	/** @start: Start address of the allocated block. */
	// 内存地址的偏移量
	u64 start;
	/** @size: Size of the allocated block. */
	// 内存的大小
	u64 size;
	/* private: */
	// 指向drm_mm
	struct drm_mm *mm;
	struct list_head node_list;
	struct list_head hole_stack;
	struct rb_node rb;
	struct rb_node rb_hole_size;
	struct rb_node rb_hole_addr;
	u64 __subtree_last;
	u64 hole_size;
	u64 subtree_max_hole;
	unsigned long flags;
#define DRM_MM_NODE_ALLOCATED_BIT	0
#define DRM_MM_NODE_SCANNED_BIT		1
#ifdef CONFIG_DRM_DEBUG_MM
	depot_stack_handle_t stack;
#endif
};
```

`struct drm_device`的`vma_offset_manager`用来分配和管理设备内存对象的虚拟地址范围，`drm_vma_offset_manager`结构体定义如下，即通过`drm_mm`来进行管理。

```c
// include/drm/drm_vma_manager.h
struct drm_vma_offset_manager {
	rwlock_t vm_lock;
	struct drm_mm vm_addr_space_mm;
};
```

drm使用drmres来管理内存资源分配，分配的资源链接到drm_device->managed.resources

```c
// include/drm/drm_managed.h
typedef void (*drmres_release_t)(struct drm_device *dev, void *res);

struct drmres_node {
	// 分配的资源链接到drm_device->managed.resources
	struct list_head	entry;
	// 资源释放回调函数
	drmres_release_t	release;
	// 资源名称
	const char		*name;
	// 资源大小
	size_t			size;
};

struct drmres {
	struct drmres_node		node;
	/*
	 * Some archs want to perform DMA into kmalloc caches
	 * and need a guaranteed alignment larger than
	 * the alignment of a 64-bit integer.
	 * Thus we use ARCH_DMA_MINALIGN for data[] which will force the same
	 * alignment for struct drmres when allocated by kmalloc().
	 */
	// 存放分配的内存资源
	u8 __aligned(ARCH_DMA_MINALIGN) data[];
};
```

**drmm的API**：

1. 分配资源，分配的资源数据存储在`drmres->data`中，并返回它

	```c
	void *drmm_kmalloc(struct drm_device *dev, size_t size, gfp_t gfp);
	void *drmm_kzalloc(struct drm_device *dev, size_t size, gfp_t gfp);
	void *drmm_kmalloc_array(struct drm_device *dev,
				       size_t n, size_t size, gfp_t flags);
	void *drmm_kcalloc(struct drm_device *dev,
				 size_t n, size_t size, gfp_t flags);
	```

2. 为drm_device添加资源释放回调函数

	```c
	#define drmm_add_action(dev, action, data) \
	__drmm_add_action(dev, action, data, #action)

	int __must_check __drmm_add_action(struct drm_device *dev,
				   drmres_release_t action,
				   void *data, const char *name);
	```

3. 释放资源，遍历`drm_device->managed.resources`来进行查找释放

	```c
	void drmm_kfree(struct drm_device *dev, void *data);
	```

### 显存管理

在drm驱动初始化对connector的显示模式进行探测并找到一个最佳的显示模式之后，就会调用驱动的`fb_helper->funcs->fb_probe`接口来创建framebuffer显存，其中涉及到的结构体有`struct drm_gem_object`，该结构体使用`drm_gem_object_init()`进行初始化

```c
// include/drm/drm_gem.h
/**
 * struct drm_gem_object - GEM buffer object
 *
 * This structure defines the generic parts for GEM buffer objects, which are
 * mostly around handling mmap and userspace handles.
 *
 * Buffer objects are often abbreviated to BO.
 */
struct drm_gem_object {
	/**
	 * @refcount:
	 *
	 * Reference count of this object
	 *
	 * Please use drm_gem_object_get() to acquire and drm_gem_object_put_locked()
	 * or drm_gem_object_put() to release a reference to a GEM
	 * buffer object.
	 */
	// 引用计数
	struct kref refcount;

	/**
	 * @handle_count:
	 *
	 * This is the GEM file_priv handle count of this object.
	 *
	 * Each handle also holds a reference. Note that when the handle_count
	 * drops to 0 any global names (e.g. the id in the flink namespace) will
	 * be cleared.
	 *
	 * Protected by &drm_device.object_name_lock.
	 */
	unsigned handle_count;

	/**
	 * @dev: DRM dev this object belongs to.
	 */
	// 指向对应的drm_device
	struct drm_device *dev;

	/**
	 * @filp:
	 *
	 * SHMEM file node used as backing storage for swappable buffer objects.
	 * GEM also supports driver private objects with driver-specific backing
	 * storage (contiguous DMA memory, special reserved blocks). In this
	 * case @filp is NULL.
	 */
	// 指向创建的shmem文件节点
	struct file *filp;

	/**
	 * @vma_node:
	 *
	 * Mapping info for this object to support mmap. Drivers are supposed to
	 * allocate the mmap offset using drm_gem_create_mmap_offset(). The
	 * offset itself can be retrieved using drm_vma_node_offset_addr().
	 *
	 * Memory mapping itself is handled by drm_gem_mmap(), which also checks
	 * that userspace is allowed to access the object.
	 */
	struct drm_vma_offset_node vma_node;

	/**
	 * @size:
	 *
	 * Size of the object, in bytes.  Immutable over the object's
	 * lifetime.
	 */
	// framebuffer的大小
	size_t size;

	/**
	 * @name:
	 *
	 * Global name for this object, starts at 1. 0 means unnamed.
	 * Access is covered by &drm_device.object_name_lock. This is used by
	 * the GEM_FLINK and GEM_OPEN ioctls.
	 */
	int name;

	/**
	 * @dma_buf:
	 *
	 * dma-buf associated with this GEM object.
	 *
	 * Pointer to the dma-buf associated with this gem object (either
	 * through importing or exporting). We break the resulting reference
	 * loop when the last gem handle for this object is released.
	 *
	 * Protected by &drm_device.object_name_lock.
	 */
	struct dma_buf *dma_buf;

	/**
	 * @import_attach:
	 *
	 * dma-buf attachment backing this object.
	 *
	 * Any foreign dma_buf imported as a gem object has this set to the
	 * attachment point for the device. This is invariant over the lifetime
	 * of a gem object.
	 *
	 * The &drm_gem_object_funcs.free callback is responsible for
	 * cleaning up the dma_buf attachment and references acquired at import
	 * time.
	 *
	 * Note that the drm gem/prime core does not depend upon drivers setting
	 * this field any more. So for drivers where this doesn't make sense
	 * (e.g. virtual devices or a displaylink behind an usb bus) they can
	 * simply leave it as NULL.
	 */
	struct dma_buf_attachment *import_attach;

	/**
	 * @resv:
	 *
	 * Pointer to reservation object associated with the this GEM object.
	 *
	 * Normally (@resv == &@_resv) except for imported GEM objects.
	 */
	struct dma_resv *resv;

	/**
	 * @_resv:
	 *
	 * A reservation object for this GEM object.
	 *
	 * This is unused for imported GEM objects.
	 */
	struct dma_resv _resv;

	/**
	 * @gpuva:
	 *
	 * Provides the list of GPU VAs attached to this GEM object.
	 *
	 * Drivers should lock list accesses with the GEMs &dma_resv lock
	 * (&drm_gem_object.resv) or a custom lock if one is provided.
	 */
	struct {
		struct list_head list;

#ifdef CONFIG_LOCKDEP
		struct lockdep_map *lock_dep_map;
#endif
	} gpuva;

	/**
	 * @funcs:
	 *
	 * Optional GEM object functions. If this is set, it will be used instead of the
	 * corresponding &drm_driver GEM callbacks.
	 *
	 * New drivers should use this.
	 *
	 */
	// 显存管理相关的接口函数
	const struct drm_gem_object_funcs *funcs;

	/**
	 * @lru_node:
	 *
	 * List node in a &drm_gem_lru.
	 */
	struct list_head lru_node;

	/**
	 * @lru:
	 *
	 * The current LRU list that the GEM object is on.
	 */
	struct drm_gem_lru *lru;
};
```

显存管理相关的接口函数

```c
/**
 * struct drm_gem_object_funcs - GEM object functions
 */
struct drm_gem_object_funcs {
	/**
	 * @free:
	 *
	 * Deconstructor for drm_gem_objects.
	 *
	 * This callback is mandatory.
	 */
	void (*free)(struct drm_gem_object *obj);

	/**
	 * @open:
	 *
	 * Called upon GEM handle creation.
	 *
	 * This callback is optional.
	 */
	int (*open)(struct drm_gem_object *obj, struct drm_file *file);

	/**
	 * @close:
	 *
	 * Called upon GEM handle release.
	 *
	 * This callback is optional.
	 */
	void (*close)(struct drm_gem_object *obj, struct drm_file *file);

	/**
	 * @print_info:
	 *
	 * If driver subclasses struct &drm_gem_object, it can implement this
	 * optional hook for printing additional driver specific info.
	 *
	 * drm_printf_indent() should be used in the callback passing it the
	 * indent argument.
	 *
	 * This callback is called from drm_gem_print_info().
	 *
	 * This callback is optional.
	 */
	void (*print_info)(struct drm_printer *p, unsigned int indent,
			   const struct drm_gem_object *obj);

	/**
	 * @export:
	 *
	 * Export backing buffer as a &dma_buf.
	 * If this is not set drm_gem_prime_export() is used.
	 *
	 * This callback is optional.
	 */
	struct dma_buf *(*export)(struct drm_gem_object *obj, int flags);

	/**
	 * @pin:
	 *
	 * Pin backing buffer in memory. Used by the drm_gem_map_attach() helper.
	 *
	 * This callback is optional.
	 */
	int (*pin)(struct drm_gem_object *obj);

	/**
	 * @unpin:
	 *
	 * Unpin backing buffer. Used by the drm_gem_map_detach() helper.
	 *
	 * This callback is optional.
	 */
	void (*unpin)(struct drm_gem_object *obj);

	/**
	 * @get_sg_table:
	 *
	 * Returns a Scatter-Gather table representation of the buffer.
	 * Used when exporting a buffer by the drm_gem_map_dma_buf() helper.
	 * Releasing is done by calling dma_unmap_sg_attrs() and sg_free_table()
	 * in drm_gem_unmap_buf(), therefore these helpers and this callback
	 * here cannot be used for sg tables pointing at driver private memory
	 * ranges.
	 *
	 * See also drm_prime_pages_to_sg().
	 */
	struct sg_table *(*get_sg_table)(struct drm_gem_object *obj);

	/**
	 * @vmap:
	 *
	 * Returns a virtual address for the buffer. Used by the
	 * drm_gem_dmabuf_vmap() helper.
	 *
	 * This callback is optional.
	 */
	int (*vmap)(struct drm_gem_object *obj, struct iosys_map *map);

	/**
	 * @vunmap:
	 *
	 * Releases the address previously returned by @vmap. Used by the
	 * drm_gem_dmabuf_vunmap() helper.
	 *
	 * This callback is optional.
	 */
	void (*vunmap)(struct drm_gem_object *obj, struct iosys_map *map);

	/**
	 * @mmap:
	 *
	 * Handle mmap() of the gem object, setup vma accordingly.
	 *
	 * This callback is optional.
	 *
	 * The callback is used by both drm_gem_mmap_obj() and
	 * drm_gem_prime_mmap().  When @mmap is present @vm_ops is not
	 * used, the @mmap callback must set vma->vm_ops instead.
	 */
	int (*mmap)(struct drm_gem_object *obj, struct vm_area_struct *vma);

	/**
	 * @evict:
	 *
	 * Evicts gem object out from memory. Used by the drm_gem_object_evict()
	 * helper. Returns 0 on success, -errno otherwise.
	 *
	 * This callback is optional.
	 */
	int (*evict)(struct drm_gem_object *obj);

	/**
	 * @status:
	 *
	 * The optional status callback can return additional object state
	 * which determines which stats the object is counted against.  The
	 * callback is called under table_lock.  Racing against object status
	 * change is "harmless", and the callback can expect to not race
	 * against object destruction.
	 *
	 * Called by drm_show_memory_stats().
	 */
	enum drm_gem_object_status (*status)(struct drm_gem_object *obj);

	/**
	 * @vm_ops:
	 *
	 * Virtual memory operations used with mmap.
	 *
	 * This is optional but necessary for mmap support.
	 */
	const struct vm_operations_struct *vm_ops;
};
```

## vblank介绍

vblank(vertical blank)是一个与显示同步相关的概念，主要用于管理和处理显示刷新（帧同步）事件，以支持平滑的图像渲染和动画显示

vblank指的是显示硬件在一次刷新完成后进入下一次刷新开始前的时间间隔，成为垂直消隐间隔，在这段时间内，显示控制器不会更新屏幕内容，这为更新显示缓冲区提供了一个无闪烁的时机。

## drm mode config

linux内核使用`struct drm_mode_config`来描述显示模式配置信息，`drm_mode_config`的主要功能之一是提供对显示器模式的管理和配置，

## drm modeset lock

## DPMS

在 Linux 内核的 DRM（Direct Rendering Manager）框架 中，DPMS（Display Power Management Signaling）是一种电源管理协议，旨在通过控制显示设备（如显示器）的电源状态来节省能源。DPMS 协议通过一组标准化的电源状态来控制显示器的开关、待机、休眠等状态，从而降低功耗。

## legacy fb兼容

在drm框架中，`struct drm_fb_helper`用于支持framebuffer兼容，主要帮助传统的帧缓冲区应用程序和现代的drm驱动程序进行交互，允许这些应用程序继续工作，同时让drm管理显示资源。

`drm_fb_helper_surface_size`结构体在DRM框架中用于帮助帧`drm_fb_helper`管理与配置帧缓冲区表面尺寸相关的参数。这个结构体主要用于处理帧缓冲区的创建和调整，确保它符合显示硬件的要求以及用户空间应用程序的需求。

## drm atomic 原子显示刷新

在drm框架中，atomic（原子模式设置）是为了改进传统的模式设置(modeset)和显示配置机制而引入的一种新模式。通过原子模式设置，显示设备的状态更容易以一种一致、可控的方式进行更改。

atomic表示一组显示配置更改要么全部成功应用，要么完全不应用，确保状态一致性，它是一种事务式的显示状态管理方法，主要应用在显示引擎的模式设置和显示缓冲更新等操作

atomic的操作主要是通过`drm_atomic_state`这个结构体来实现的，前期的工作主要是更新`drm_atomic_state`中对应crtc plane connector的显示模式，之后再调用`drm_mode_config->funcs->atomic_commit`进行提交

### drm_atomic的核心数据结构

`struct drm_atomic_state`

```c
// include/drm/drm_atomic.h
/**
 * struct drm_atomic_state - the global state object for atomic updates
 * @ref: count of all references to this state (will not be freed until zero)
 * @dev: parent DRM device
 * @async_update: hint for asynchronous plane update
 * @planes: pointer to array of structures with per-plane data
 * @crtcs: pointer to array of CRTC pointers
 * @num_connector: size of the @connectors and @connector_states arrays
 * @connectors: pointer to array of structures with per-connector data
 * @num_private_objs: size of the @private_objs array
 * @private_objs: pointer to array of private object pointers
 * @acquire_ctx: acquire context for this atomic modeset state update
 *
 * States are added to an atomic update by calling drm_atomic_get_crtc_state(),
 * drm_atomic_get_plane_state(), drm_atomic_get_connector_state(), or for
 * private state structures, drm_atomic_get_private_obj_state().
 */
struct drm_atomic_state {
	struct kref ref;

	struct drm_device *dev;

	/**
	 * @allow_modeset:
	 *
	 * Allow full modeset. This is used by the ATOMIC IOCTL handler to
	 * implement the DRM_MODE_ATOMIC_ALLOW_MODESET flag. Drivers should
	 * never consult this flag, instead looking at the output of
	 * drm_atomic_crtc_needs_modeset().
	 */
	bool allow_modeset : 1;
	/**
	 * @legacy_cursor_update:
	 *
	 * Hint to enforce legacy cursor IOCTL semantics.
	 *
	 * WARNING: This is thoroughly broken and pretty much impossible to
	 * implement correctly. Drivers must ignore this and should instead
	 * implement &drm_plane_helper_funcs.atomic_async_check and
	 * &drm_plane_helper_funcs.atomic_async_commit hooks. New users of this
	 * flag are not allowed.
	 */
	bool legacy_cursor_update : 1;
	bool async_update : 1;
	/**
	 * @duplicated:
	 *
	 * Indicates whether or not this atomic state was duplicated using
	 * drm_atomic_helper_duplicate_state(). Drivers and atomic helpers
	 * should use this to fixup normal  inconsistencies in duplicated
	 * states.
	 */
	bool duplicated : 1;
	// 记录plane crtc connector的状态
	struct __drm_planes_state *planes;
	struct __drm_crtcs_state *crtcs;
	int num_connector;
	struct __drm_connnectors_state *connectors;
	int num_private_objs;
	struct __drm_private_objs_state *private_objs;

	struct drm_modeset_acquire_ctx *acquire_ctx;

	/**
	 * @fake_commit:
	 *
	 * Used for signaling unbound planes/connectors.
	 * When a connector or plane is not bound to any CRTC, it's still important
	 * to preserve linearity to prevent the atomic states from being freed to early.
	 *
	 * This commit (if set) is not bound to any CRTC, but will be completed when
	 * drm_atomic_helper_commit_hw_done() is called.
	 */
	struct drm_crtc_commit *fake_commit;

	/**
	 * @commit_work:
	 *
	 * Work item which can be used by the driver or helpers to execute the
	 * commit without blocking.
	 */
	struct work_struct commit_work;
};

struct __drm_planes_state {
	struct drm_plane *ptr;
	struct drm_plane_state *state, *old_state, *new_state;
};

struct __drm_crtcs_state {
	struct drm_crtc *ptr;
	struct drm_crtc_state *state, *old_state, *new_state;

	/**
	 * @commit:
	 *
	 * A reference to the CRTC commit object that is kept for use by
	 * drm_atomic_helper_wait_for_flip_done() after
	 * drm_atomic_helper_commit_hw_done() is called. This ensures that a
	 * concurrent commit won't free a commit object that is still in use.
	 */
	struct drm_crtc_commit *commit;

	s32 __user *out_fence_ptr;
	u64 last_vblank_count;
};

struct __drm_connnectors_state {
	struct drm_connector *ptr;
	struct drm_connector_state *state, *old_state, *new_state;
	/**
	 * @out_fence_ptr:
	 *
	 * User-provided pointer which the kernel uses to return a sync_file
	 * file descriptor. Used by writeback connectors to signal completion of
	 * the writeback.
	 */
	s32 __user *out_fence_ptr;
};
```

## drm驱动框架源码

### drm core层初始化代码

内核在启动时调用`drm_core_init()`进行drm驱动的初始化工作

```c
// drivers/gpu/drm/drm_drv.c
static int __init drm_core_init(void)
{
	int ret;

	drm_connector_ida_init();
	idr_init(&drm_minors_idr);
	drm_memcpy_init_early();

	ret = drm_sysfs_init();
	if (ret < 0) {
		DRM_ERROR("Cannot create DRM class: %d\n", ret);
		goto error;
	}

	drm_debugfs_root = debugfs_create_dir("dri", NULL);

	ret = register_chrdev(DRM_MAJOR, "drm", &drm_stub_fops);
	if (ret < 0)
		goto error;

	ret = accel_core_init();
	if (ret < 0)
		goto error;

	drm_privacy_screen_lookup_init();

	drm_core_init_complete = true;

	DRM_DEBUG("Initialized\n");
	return 0;

error:
	drm_core_exit();
	return ret;
}
```

![](https://raw.githubusercontent.com/JackHuang021/images/master/linux_drm_core_init.png)

### 3.2 drm_device的内存资源管理

drm通过`drm_device->managed`来管理drm驱动初始化过程中分配的内存资源，具体的源码位置位于`drivers/gpu/drm/drm_managed.c`，分配内存时使用`drmm_xxx()`接口便可将分配的内存资源通过`managed`管理起来

### drm master 和 drm_auth

在 Linux DRM（Direct Rendering Manager）子系统中，drm_master 和 drm_auth 是与 认证（Authentication） 相关的一项机制，主要用于确保只有经过认证的用户或进程才能访问和操作图形硬件。这种认证机制用于保障图形设备的安全性和资源访问的合理性，避免未授权的进程或用户直接控制图形硬件，造成安全隐患或者资源冲突。

#### `struct drm_file` 结构体

drm_file 结构体在 Linux DRM (Direct Rendering Manager) 子系统中是用于表示与图形设备交互的“文件句柄”。它充当了用户空间进程与内核之间的中介，保存着与特定图形设备操作相关的信息，特别是与设备资源（如缓冲区、IOCTL 操作、上下文管理等）的访问权限相关的信息。每当用户进程打开一个图形设备时，内核会为其分配一个 drm_file 结构体，用来跟踪该进程对设备的访问。

```c
// include/drm/drm_file.h
/**
 * struct drm_file - DRM file private data
 *
 * This structure tracks DRM state per open file descriptor.
 */
struct drm_file {
	/**
	 * @authenticated:
	 *
	 * Whether the client is allowed to submit rendering, which for legacy
	 * nodes means it must be authenticated.
	 *
	 * See also the :ref:`section on primary nodes and authentication
	 * <drm_primary_node>`.
	 */
	/** 表示当前用户进程获得了drm master权限 */
	bool authenticated;

	/**
	 * @stereo_allowed:
	 *
	 * True when the client has asked us to expose stereo 3D mode flags.
	 */
	bool stereo_allowed;

	/**
	 * @universal_planes:
	 *
	 * True if client understands CRTC primary planes and cursor planes
	 * in the plane list. Automatically set when @atomic is set.
	 */
	bool universal_planes;

	/** @atomic: True if client understands atomic properties. */
	bool atomic;

	/**
	 * @aspect_ratio_allowed:
	 *
	 * True, if client can handle picture aspect ratios, and has requested
	 * to pass this information along with the mode.
	 */
	bool aspect_ratio_allowed;

	/**
	 * @writeback_connectors:
	 *
	 * True if client understands writeback connectors
	 */
	bool writeback_connectors;

	/**
	 * @was_master:
	 *
	 * This client has or had, master capability. Protected by struct
	 * &drm_device.master_mutex.
	 *
	 * This is used to ensure that CAP_SYS_ADMIN is not enforced, if the
	 * client is or was master in the past.
	 */
	/** 表示当前进程具有drm master权限 */
	bool was_master;

	/**
	 * @is_master:
	 *
	 * This client is the creator of @master. Protected by struct
	 * &drm_device.master_mutex.
	 *
	 * See also the :ref:`section on primary nodes and authentication
	 * <drm_primary_node>`.
	 */
	/** 表示当前进程为drm_master的创建者 */
	bool is_master;

	/**
	 * @supports_virtualized_cursor_plane:
	 *
	 * This client is capable of handling the cursor plane with the
	 * restrictions imposed on it by the virtualized drivers.
	 *
	 * This implies that the cursor plane has to behave like a cursor
	 * i.e. track cursor movement. It also requires setting of the
	 * hotspot properties by the client on the cursor plane.
	 */
	bool supports_virtualized_cursor_plane;

	/**
	 * @master:
	 *
	 * Master this node is currently associated with. Protected by struct
	 * &drm_device.master_mutex, and serialized by @master_lookup_lock.
	 *
	 * Only relevant if drm_is_primary_client() returns true. Note that
	 * this only matches &drm_device.master if the master is the currently
	 * active one.
	 *
	 * To update @master, both &drm_device.master_mutex and
	 * @master_lookup_lock need to be held, therefore holding either of
	 * them is safe and enough for the read side.
	 *
	 * When dereferencing this pointer, either hold struct
	 * &drm_device.master_mutex for the duration of the pointer's use, or
	 * use drm_file_get_master() if struct &drm_device.master_mutex is not
	 * currently held and there is no other need to hold it. This prevents
	 * @master from being freed during use.
	 *
	 * See also @authentication and @is_master and the :ref:`section on
	 * primary nodes and authentication <drm_primary_node>`.
	 */
	/** 指向当前创建的drm_master */
	struct drm_master *master;

	/** @master_lookup_lock: Serializes @master. */
	spinlock_t master_lookup_lock;

	/**
	 * @pid: Process that is using this file.
	 *
	 * Must only be dereferenced under a rcu_read_lock or equivalent.
	 *
	 * Updates are guarded with dev->filelist_mutex and reference must be
	 * dropped after a RCU grace period to accommodate lockless readers.
	 */
	struct pid __rcu *pid;

	/** @client_id: A unique id for fdinfo */
	u64 client_id;

	/** @magic: Authentication magic, see @authenticated. */
	drm_magic_t magic;

	/**
	 * @lhead:
	 *
	 * List of all open files of a DRM device, linked into
	 * &drm_device.filelist. Protected by &drm_device.filelist_mutex.
	 */
	struct list_head lhead;

	/** @minor: &struct drm_minor for this file. */
	struct drm_minor *minor;

	/**
	 * @object_idr:
	 *
	 * Mapping of mm object handles to object pointers. Used by the GEM
	 * subsystem. Protected by @table_lock.
	 */
	struct idr object_idr;

	/** @table_lock: Protects @object_idr. */
	spinlock_t table_lock;

	/** @syncobj_idr: Mapping of sync object handles to object pointers. */
	struct idr syncobj_idr;
	/** @syncobj_table_lock: Protects @syncobj_idr. */
	spinlock_t syncobj_table_lock;

	/** @filp: Pointer to the core file structure. */
	struct file *filp;

	/**
	 * @driver_priv:
	 *
	 * Optional pointer for driver private data. Can be allocated in
	 * &drm_driver.open and should be freed in &drm_driver.postclose.
	 */
	void *driver_priv;

	/**
	 * @fbs:
	 *
	 * List of &struct drm_framebuffer associated with this file, using the
	 * &drm_framebuffer.filp_head entry.
	 *
	 * Protected by @fbs_lock. Note that the @fbs list holds a reference on
	 * the framebuffer object to prevent it from untimely disappearing.
	 */
	struct list_head fbs;

	/** @fbs_lock: Protects @fbs. */
	struct mutex fbs_lock;

	/**
	 * @blobs:
	 *
	 * User-created blob properties; this retains a reference on the
	 * property.
	 *
	 * Protected by @drm_mode_config.blob_lock;
	 */
	struct list_head blobs;

	/** @event_wait: Waitqueue for new events added to @event_list. */
	wait_queue_head_t event_wait;

	/**
	 * @pending_event_list:
	 *
	 * List of pending &struct drm_pending_event, used to clean up pending
	 * events in case this file gets closed before the event is signalled.
	 * Uses the &drm_pending_event.pending_link entry.
	 *
	 * Protect by &drm_device.event_lock.
	 */
	struct list_head pending_event_list;

	/**
	 * @event_list:
	 *
	 * List of &struct drm_pending_event, ready for delivery to userspace
	 * through drm_read(). Uses the &drm_pending_event.link entry.
	 *
	 * Protect by &drm_device.event_lock.
	 */
	struct list_head event_list;

	/**
	 * @event_space:
	 *
	 * Available event space to prevent userspace from
	 * exhausting kernel memory. Currently limited to the fairly arbitrary
	 * value of 4KB.
	 */
	int event_space;

	/** @event_read_lock: Serializes drm_read(). */
	struct mutex event_read_lock;

	/**
	 * @prime:
	 *
	 * Per-file buffer caches used by the PRIME buffer sharing code.
	 */
	struct drm_prime_file_private prime;

	/* private: */
#if IS_ENABLED(CONFIG_DRM_LEGACY)
	unsigned long lock_count; /* DRI1 legacy lock count */
#endif
};
```

#### `struct drm_auth` 结构体

```c
// include/uapi/drm/drm.h
/*
 * DRM_IOCTL_GET_MAGIC and DRM_IOCTL_AUTH_MAGIC ioctl argument type.
 */
struct drm_auth {
	drm_magic_t magic;
};

```

#### DRM 认证的工作原理

1. **认证标识**：每个打开图形设备的进程都会分配一个 `drm_file` 结构体，认证标识为`drm_file.authenticated`字段，标识该进程是否经过了认证，如果认证失败，将无法执行进一步的图形操作。注意root用户创建的进程访问DRM设备，`drm_file.authenticated`初始化的时候就置为true了。
2. **主会话与认证**：在多个进程访问DRM设备时，其中一个进程通常会被指定为主进程，通过`drm_file.is_master`字段进行判断是否为主会话，主会话拥有DRM设备的完全控制权限，其他会话则需要通过认证才能执行特定操作。
3. **认证过程**：用户进程 使用 `DRM_IOCTL_GET_MAGIC` ioctl 获取一个 32bit 整型的 token，然后必须要drm_master 进程使用 `DRM_IOCTL_AUTH_MAGIC` ioctl 来将token传入到DRM驱动进行认证后，才能将该用户进程的`drm_file.authenticated`置为`true`。
4. **需要认证的ioctl**：涉及到硬件操作的ioctl一般都是需要`DRM_AUTH`权限的，具体参考`drivers/gpu/drm/drm_ioctl.c`中的`drm_ioctls`，需要认证的ioctl会带上`DRM_AUTH`标志，执行的时候会检查`drm_file.authenicated`字段，为false的话直接退出了。

```c
DRM_IOCTL_DEF(DRM_IOCTL_GET_MAGIC, drm_getmagic, 0),
DRM_IOCTL_DEF(DRM_IOCTL_AUTH_MAGIC, drm_authmagic, DRM_MASTER),

int drm_getmagic(struct drm_device *dev, void *data, struct drm_file *file_priv)
{
	struct drm_auth *auth = data;
	int ret = 0;

	mutex_lock(&dev->master_mutex);
	if (!file_priv->magic) {
		ret = idr_alloc(&file_priv->master->magic_map, file_priv,
				1, 0, GFP_KERNEL);
		if (ret >= 0)
			file_priv->magic = ret;
	}
	auth->magic = file_priv->magic;
	mutex_unlock(&dev->master_mutex);

	drm_dbg_core(dev, "%u\n", auth->magic);

	return ret < 0 ? ret : 0;
}

int drm_authmagic(struct drm_device *dev, void *data,
		  struct drm_file *file_priv)
{
	struct drm_auth *auth = data;
	struct drm_file *file;

	drm_dbg_core(dev, "%u\n", auth->magic);

	mutex_lock(&dev->master_mutex);
	file = idr_find(&file_priv->master->magic_map, auth->magic);
	if (file) {
		file->authenticated = 1;
		idr_replace(&file_priv->master->magic_map, NULL, auth->magic);
	}
	mutex_unlock(&dev->master_mutex);

	return file ? 0 : -EINVAL;
}
```

DRM认证过程：
![](https://raw.githubusercontent.com/JackHuang021/images/master/20250227163719.png)

```bash
8f86c82aba8b13e732cfdd6d0e19a7dd48197e43 drm/connector: demote connector force-probes for non-master clients

869e76f7a918f010bd4q518d58886969b1f642a04 drm: avoid circular locks in drm_mode_getconnector
```



## 3 Phytium E2000/X100 DRM DC驱动

### 3.1 phytium E2000 DC控制器设备树描述

```c
dc0: dc@32000000 {
	compatible = "phytium,dc";
	reg = <0x0 0x32000000 0x0 0x8000>;
	interrupts = <GIC_SPI 44 IRQ_TYPE_LEVEL_HIGH>;
	pipe_mask = /bits/ 8 <0x3>;
	edp_mask = /bits/ 8 <0x0>;
	status = "okay";
};
```

### 3.2 硬件抽象

1. DC控制器：CRTC, Plane
2. DP：Encoder, Connector

### 3.3 phytium drm驱动中的结构体

#### 3.3.1 struct phytium_display_private

```c
// drivers/gpu/drm/phytium/phytium_display_drv.h
struct phytium_display_private {
	/* hw */
	void __iomem *regs;
	void __iomem *vram_addr;
	struct phytium_device_info info;
	// 记录当前支持的显存类型
	char support_memory_type;
	char reserve[3];
	/* DC寄存器基地址，目前E2000有两路DC */
	uint32_t dc_reg_base[3];
	uint32_t dcreq_reg_base[3];
	uint32_t address_transform_base;
	uint32_t phy_access_base[3];

	/* drm */
	struct drm_device *dev;
	int irq;

	/* fb_dev */
	struct drm_fb_helper fbdev_helper;
	struct phytium_gem_object *fbdev_phytium_gem;

	int save_reg[3];
	// 用于链接phytium_gem_object
	struct list_head gem_list_head;

	struct work_struct hotplug_work;
	spinlock_t hotplug_irq_lock;

	void (*vramm_hw_init)(struct phytium_display_private *priv);
	void (*display_shutdown)(struct drm_device *dev);
	int (*display_pm_suspend)(struct drm_device *dev);
	int (*display_pm_resume)(struct drm_device *dev);
	void (*dc_hw_clear_msi_irq)(struct phytium_display_private *priv,
								uint32_t phys_pipe);
	int (*dc_hw_fb_format_check)(const struct drm_mode_fb_cmd2 *mode_cmd,
								 int count);

	struct gen_pool *memory_pool;
	resource_size_t pool_phys_addr;
	resource_size_t pool_size;
	void *pool_virt_addr;
	// 记录当前的显存使用大小
	uint64_t mem_state[PHYTIUM_MEM_STATE_TYPE_COUNT];

	int dma_inited;
	struct dma_chan *dam_chan;
};
```

#### 3.3.2 struct phytium_device_info结构体定义

```c
struct phytium_device_info {
	/* 硬件平台的标志，目前定义了X100和E2000 */
	unsigned char platform_mask;
	unsigned char pipe_mask;
	unsigned char num_pipes;
	unsigned char total_pipes;
	unsigned char edp_mask;
	unsigned int crtc_clock_max;
	unsigned int hdisplay_max;
	unsigned int vdisplay_max;
	unsigned int backlight_max;
	unsigned long address_mask;
};
```

#### 3.3.3 struct phytium_dp_device结构体定义

`struct phytium_dp_device`封装了`struct drm_encoder`和`struct drm_connector`

```c
struct phytium_dp_device {
	// 指向struct drm_device
	struct drm_device *dev;
	struct drm_encoder encoder;
	struct drm_connector connector;
	// 对应的显示接口
	int port;
	struct drm_display_mode mode;
	bool link_trained;
	bool detect_done;
	// 是否为edp接口
	bool is_edp;
	bool reserve0;
	struct drm_dp_aux aux;
	unsigned char dpcd[DP_RECEIVE_CAP_SIZE];
	uint8_t edp_dpcd[EDP_DISPLAY_CTL_CAP_SIZE];
	unsigned char downstream_ports[DP_MAX_DOWNSTREAM_PORTS];
	unsigned char sink_count;

	// dp链路速率，飞腾E2000为1.62Gbps/lane 2.7Gbps/lane 5.4Gbps/lane 8.1Gbps/lane
	int *source_rates;
	// source和sink支持的速率
	int num_source_rates;
	int sink_rates[DP_MAX_SUPPORTED_RATES];
	int num_sink_rates;
	int common_rates[DP_MAX_SUPPORTED_RATES];
	// source和sink共同支持的速率个数
	int num_common_rates;

	int source_max_lane_count;
	int sink_max_lane_count;
	int common_max_lane_count;

	// 当前支持的最大速率
	int max_link_rate;
	// 当前匹配的lane最大数量
	int max_link_lane_count;
	int link_rate;
	int link_lane_count;
	struct work_struct train_retry_work;
	int train_retry_count;
	uint32_t trigger_train_fail;


	unsigned char train_set[4];
	struct edid *edp_edid;
	bool has_audio;
	bool fast_train_support;
	bool hw_spread_enable;
	bool reserve[1];
	struct platform_device *audio_pdev;
	struct audio_info audio_info;
	hdmi_codec_plugged_cb plugged_cb;
	struct device *codev_dev;
	struct phytium_dp_compliance compliance;
	struct phytium_dp_func *funcs;
	struct phytium_dp_hpd_state dp_hpd_state;

	struct phytium_panel panel;
	struct drm_display_mode native_mode;
};
```

`struct phytium_dp_hpd_state`结构体，HDP即Hot Plug Detect

```c
struct phytium_dp_hpd_state {
	// 热插拔连接或者断开事件中断状态
	bool hpd_event_state;
	// 热插拔 irq中断
	bool hpd_irq_state;
	// 控制器HPD输入端口的raw state
	bool hpd_raw_state;
	// 
	bool hpd_irq_enable;
};
```

#### 3.3.4 phytium crtc相关结构体定义

```c
// drivers/gpu/drm/phytium/phytium_crtc.h
struct phytium_crtc {
	struct drm_crtc base;
	int phys_pipe;
	unsigned int bpc;

	uint32_t src_width;
	uint32_t src_height;
	uint32_t dst_width;
	uint32_t dst_height; 
	uint32_t dst_x;
	uint32_t dst_y;
	bool scale_enable;
	bool reserve[3];

	// 像素时钟寄存器配置
	void (*dc_hw_config_pix_clock)(struct drm_crtc *crtc, int clock);
	void (*dc_hw_disable)(struct drm_crtc *crtc);
	void (*dc_hw_reset)(struct drm_crtc *crtc);
};
```

#### 3.3.5 phytium_plane结构体定义

```c
// drivers/gpu/drm/phytium/phytium_plane.h
struct phytium_plane {
	struct drm_plane base;
	/* CRTC的索引号 */
	int phys_pipe;
	// framebuffer数据起始地址
	unsigned long iova[PHYTIUM_FORMAT_MAX_PLANE];
	unsigned long size[PHYTIUM_FORMAT_MAX_PLANE];
	unsigned int format;
	unsigned int tiling[PHYTIUM_FORMAT_MAX_PLANE];
	unsigned int swizzle;
	unsigned int uv_swizzle;
	unsigned int rot_angle;

	/* only for cursor */
	bool enable;
	bool reserve[3];
	unsigned int cursor_x;
	unsigned int cursor_y;
	unsigned int cursor_hot_x;
	unsigned int cursor_hot_y;

	// 获取像素颜色格式
	void (*dc_hw_plane_get_format)(const uint64_t **format_modifiers,
								   const uint32_t **formats,
								   uint32_t *format_count);
	void (*dc_hw_update_dcreq)(struct drm_plane *plane);
	// 配置framebuffer数据起始地址的高8位
	void (*dc_hw_update_primary_hi_addr)(struct drm_plane *plane);
	void (*dc_hw_update_cursor_hi_addr)(struct drm_plane *plane, uint64_t iova);
};
```

#### struct phytium_gem_object

`phytium_gem_object`封装了一个`drm_gem_object`，用于记录显存的大小和地址

```c
// drivers/gpu/drm/phytium/phytium_gem.h
struct phytium_gem_object {
	struct drm_gem_object base;
	phys_addr_t phys_addr;
	// 存储显存的dma地址
	dma_addr_t iova;
	// 指向分配的DMA内存区域的虚拟地址
	void *vaddr;
	unsigned long size;
	struct sg_table *sgt;
	// 记录显存的类型，phytium_display_private->support_memory_type会记录支持的显存类型
	// 在phytium_gem_create_object会确定当前使用的显存类型
	char memory_type;
	char reserve[3];
	struct list_head list;
	void *vaddr_save;
};
```

#### struct phytium_framebuffer

```c
// drivers/gpu/drm/phytium/phytium_fb.h
struct phytium_framebuffer {
	struct drm_framebuffer base;
	struct phytium_gem_object *phytium_gem_obj[PHYTIUM_FORMAT_MAX_PLANE];
};
```

### dp相关的操作接口

```c
struct phytium_dp_func {
	uint8_t (*dp_hw_get_source_lane_count)(struct phytium_dp_device *phytium_dp);
	int (*dp_hw_reset)(struct phytium_dp_device *phytium_dp);
	bool (*dp_hw_spread_is_enable)(struct phytium_dp_device *phytium_dp);
	int (*dp_hw_set_backlight)(struct phytium_dp_device *phytium_dp, uint32_t level);
	uint32_t (*dp_hw_get_backlight)(struct phytium_dp_device *phytium_dp);
	void (*dp_hw_disable_backlight)(struct phytium_dp_device *phytium_dp);
	void (*dp_hw_enable_backlight)(struct phytium_dp_device *phytium_dp);
	void (*dp_hw_poweroff_panel)(struct phytium_dp_device *phytium_dp);
	void (*dp_hw_poweron_panel)(struct phytium_dp_device *phytium_dp);
	int (*dp_hw_init_phy)(struct phytium_dp_device *phytium_dp);
	void (*dp_hw_set_phy_lane_setting)(struct phytium_dp_device *phytium_dp,
					   uint32_t link_rate, uint8_t train_set);
	int (*dp_hw_set_phy_lane_and_rate)(struct phytium_dp_device *phytium_dp,
					   uint8_t link_lane_count,
					   uint32_t link_rate);
};
```

### 3.4 phytium drm_driver实例

#### 概述和定位

位置: drivers/gpu/drm/phytium_ftd330/

作用: 为飞腾自研的 FTD330 系列显示控制器提供 KMS（Kernel Mode Setting）+ GEM 图形缓冲区管理，不包含 2D/3D 加速。

规模: 约 28.5K 行 C 代码，58 个文件，按功能拆分成多个独立模块。

驱动特性开关: 通过 Kconfig 可裁剪出针对不同芯片版本（0x30b/0x310/0x311/0x31b/0x331/0x335）和不同子功能（PSU/MMU/DEC/WB/PVRIC 等）的固件组合。

运行模式: 同时支持 platform device（设备树 + ACPI）与 PCI device 两种注册方式。

#### 构建系统（Makefile + Kconfig）

Makefile 把驱动拆为几个可选项：

+ 基础 KMS：ftd330_crtc.o / ftd330_plane.o / ftd330_fb.o / ftd330_gem.o / ftd330_drv.o / ftd330_simple_enc.o 等；
+ FTD330 硬件抽象层：FTD330/ 子目录下的 ftd330_dc.o、ftd330_dc_hw.o、ftd330_dc_dec.o（压缩）以及前处理/后处理/写回三个子模块；
+ 输出接口（互斥）：
	+ 实际板级：phytium_dp.o + ftd330_dp.o + phytium_panel.o + phytium_edp_pwm.o；
	+ 仿真器：phytium_dp_emulator.o（CONFIG_PHYTIUM_DCDP_EMULATOR）；
	+ 可选子模块：PSR（phytium_psr.o）、VRR、debugfs、Bios 参数解析、SE 通信、MMU、DEC、Writeback、fbdev 兼容层、虚拟显示。

Kconfig 还暴露了 20+ 配置项，例如 PHYTIUM_PSR、PHYTIUM_DEC、PHYTIUM_PCIE、PHYTIUM_MMU、PHYTIUM_EDP_BL、PHYTIUM_LANE_TRAIN 等

#### 目录结构

```bash
phytium_ftd330/
├── Makefile / Kconfig              # 条件编译
├── ftd330_drv.c / ftd330_drv.h     # DRM 驱动入口
├── ftd330_type.h                   # 芯片能力表、共享数据结构
├── ftd330_crtc.c / ftd330_crtc.h   # CRTC 对象
├── ftd330_plane.c / ftd330_plane.h # Plane 对象
├── ftd330_simple_enc.c             # 简单编码器（MUX 选择）
├── ftd330_gem.c / ftd330_gem.h     # GEM 对象管理
├── ftd330_writeback.c              # 写回连接器
├── ftd330_virtual.c                # 虚拟显示
├── ftd330_dc_property.h            # 64 个自定义属性的宏框架
├── ftd330_debug.c                  # DebugFS 寄存器捕获
├── ftd330_dc_mmu.c                  # DC MMU 4KB 页表管理
├── ftd330_dc_pvric.c                # PVRIC 无损压缩
├── phytium_dp.c / phytium_dp.h     # DP/eDP 主驱动（5178 行）
├── phytium_dp_reg.h                # DP 寄存器定义（~25KB）
├── phytium_panel.c                 # eDP 面板上电时序
├── phytium_edp_pwm.c               # eDP 背光 PWM
├── phytium_psr.c                   # PSR / PSR2 / PSRSF
├── phytium_vrr.c                   # VRR / FreeSync
├── phytium_parse_bios.c            # BIOS/VBT 参数解析
├── dw_mipi_dsi.c                   # Synopsys DW-MIPI-DSI 桥接（未链接）
└── FTD330/                         # 硬件抽象层（HAL）
    ├── ftd330_dc.c / ftd330_dc.h   # 组件框架绑定
    ├── ftd330_dc_hw.c / ftd330_dc_hw.h  # 寄存器级 HAL
    ├── ftd330_dc_dec.c             # DEC400 压缩引擎
    ├── writeback/ftd330_dc_writeback.c  # 写回后端
    ├── preprocess/                 # 预处理器（混合层状态）
    └── postprocess/                # 提交触发器
```

```c
// drivers/gpu/drm/phytium/phytium_display_drv.c
struct drm_driver phytium_display_drm_driver = {
	.driver_features	= DRIVER_HAVE_IRQ   |
				  DRIVER_MODESET    |
				  DRIVER_ATOMIC     |
				  DRIVER_GEM,
	.load			= phytium_display_load,
	.unload			= phytium_display_unload,
	.lastclose		= drm_fb_helper_lastclose,
	.gem_prime_import	= drm_gem_prime_import,
	.gem_prime_import_sg_table = phytium_gem_prime_import_sg_table,
	.dumb_create		= phytium_gem_dumb_create,
	.ioctls			= phytium_ioctls,
	.num_ioctls		= ARRAY_SIZE(phytium_ioctls),
	.fops			= &phytium_drm_driver_fops,
	.name			= DRV_NAME,
	.desc			= DRV_DESC,
	.date			= DRV_DATE,
	.major			= DRV_MAJOR,
	.minor			= DRV_MINOR,
};
```

ioctl命令的定义
```c
#define DRM_IOCTL_BASE			'd'
#define DRM_IO(nr)			_IO(DRM_IOCTL_BASE,nr)
#define DRM_IOR(nr,type)		_IOR(DRM_IOCTL_BASE,nr,type)
#define DRM_IOW(nr,type)		_IOW(DRM_IOCTL_BASE,nr,type)
#define DRM_IOWR(nr,type)		_IOWR(DRM_IOCTL_BASE,nr,type)

#define DRM_IOCTL_VERSION		DRM_IOWR(0x00, struct drm_version)
#define DRM_IOCTL_GET_UNIQUE		DRM_IOWR(0x01, struct drm_unique)
#define DRM_IOCTL_GET_MAGIC		DRM_IOR( 0x02, struct drm_auth)
#define DRM_IOCTL_IRQ_BUSID		DRM_IOWR(0x03, struct drm_irq_busid)
#define DRM_IOCTL_GET_MAP               DRM_IOWR(0x04, struct drm_map)
#define DRM_IOCTL_GET_CLIENT            DRM_IOWR(0x05, struct drm_client)
#define DRM_IOCTL_GET_STATS             DRM_IOR( 0x06, struct drm_stats)
#define DRM_IOCTL_SET_VERSION		DRM_IOWR(0x07, struct drm_set_version)
#define DRM_IOCTL_MODESET_CTL           DRM_IOW(0x08, struct drm_modeset_ctl)
#define DRM_IOCTL_GEM_CLOSE		DRM_IOW (0x09, struct drm_gem_close)
#define DRM_IOCTL_GEM_FLINK		DRM_IOWR(0x0a, struct drm_gem_flink)
#define DRM_IOCTL_GEM_OPEN		DRM_IOWR(0x0b, struct drm_gem_open)
#define DRM_IOCTL_GET_CAP		DRM_IOWR(0x0c, struct drm_get_cap)
#define DRM_IOCTL_SET_CLIENT_CAP	DRM_IOW( 0x0d, struct drm_set_client_cap)

#define DRM_IOCTL_SET_UNIQUE		DRM_IOW( 0x10, struct drm_unique)
#define DRM_IOCTL_AUTH_MAGIC		DRM_IOW( 0x11, struct drm_auth)
#define DRM_IOCTL_BLOCK			DRM_IOWR(0x12, struct drm_block)
#define DRM_IOCTL_UNBLOCK		DRM_IOWR(0x13, struct drm_block)
#define DRM_IOCTL_CONTROL		DRM_IOW( 0x14, struct drm_control)
#define DRM_IOCTL_ADD_MAP		DRM_IOWR(0x15, struct drm_map)
#define DRM_IOCTL_ADD_BUFS		DRM_IOWR(0x16, struct drm_buf_desc)
#define DRM_IOCTL_MARK_BUFS		DRM_IOW( 0x17, struct drm_buf_desc)
#define DRM_IOCTL_INFO_BUFS		DRM_IOWR(0x18, struct drm_buf_info)
#define DRM_IOCTL_MAP_BUFS		DRM_IOWR(0x19, struct drm_buf_map)
#define DRM_IOCTL_FREE_BUFS		DRM_IOW( 0x1a, struct drm_buf_free)

#define DRM_IOCTL_RM_MAP		DRM_IOW( 0x1b, struct drm_map)

#define DRM_IOCTL_SET_SAREA_CTX		DRM_IOW( 0x1c, struct drm_ctx_priv_map)
#define DRM_IOCTL_GET_SAREA_CTX 	DRM_IOWR(0x1d, struct drm_ctx_priv_map)

#define DRM_IOCTL_SET_MASTER            DRM_IO(0x1e)
#define DRM_IOCTL_DROP_MASTER           DRM_IO(0x1f)

#define DRM_IOCTL_ADD_CTX		DRM_IOWR(0x20, struct drm_ctx)
#define DRM_IOCTL_RM_CTX		DRM_IOWR(0x21, struct drm_ctx)
#define DRM_IOCTL_MOD_CTX		DRM_IOW( 0x22, struct drm_ctx)
#define DRM_IOCTL_GET_CTX		DRM_IOWR(0x23, struct drm_ctx)
#define DRM_IOCTL_SWITCH_CTX		DRM_IOW( 0x24, struct drm_ctx)
#define DRM_IOCTL_NEW_CTX		DRM_IOW( 0x25, struct drm_ctx)
#define DRM_IOCTL_RES_CTX		DRM_IOWR(0x26, struct drm_ctx_res)
#define DRM_IOCTL_ADD_DRAW		DRM_IOWR(0x27, struct drm_draw)
#define DRM_IOCTL_RM_DRAW		DRM_IOWR(0x28, struct drm_draw)
#define DRM_IOCTL_DMA			DRM_IOWR(0x29, struct drm_dma)
#define DRM_IOCTL_LOCK			DRM_IOW( 0x2a, struct drm_lock)
#define DRM_IOCTL_UNLOCK		DRM_IOW( 0x2b, struct drm_lock)
#define DRM_IOCTL_FINISH		DRM_IOW( 0x2c, struct drm_lock)

#define DRM_IOCTL_PRIME_HANDLE_TO_FD    DRM_IOWR(0x2d, struct drm_prime_handle)
#define DRM_IOCTL_PRIME_FD_TO_HANDLE    DRM_IOWR(0x2e, struct drm_prime_handle)

#define DRM_IOCTL_AGP_ACQUIRE		DRM_IO(  0x30)
#define DRM_IOCTL_AGP_RELEASE		DRM_IO(  0x31)
#define DRM_IOCTL_AGP_ENABLE		DRM_IOW( 0x32, struct drm_agp_mode)
#define DRM_IOCTL_AGP_INFO		DRM_IOR( 0x33, struct drm_agp_info)
#define DRM_IOCTL_AGP_ALLOC		DRM_IOWR(0x34, struct drm_agp_buffer)
#define DRM_IOCTL_AGP_FREE		DRM_IOW( 0x35, struct drm_agp_buffer)
#define DRM_IOCTL_AGP_BIND		DRM_IOW( 0x36, struct drm_agp_binding)
#define DRM_IOCTL_AGP_UNBIND		DRM_IOW( 0x37, struct drm_agp_binding)

#define DRM_IOCTL_SG_ALLOC		DRM_IOWR(0x38, struct drm_scatter_gather)
#define DRM_IOCTL_SG_FREE		DRM_IOW( 0x39, struct drm_scatter_gather)

#define DRM_IOCTL_WAIT_VBLANK		DRM_IOWR(0x3a, union drm_wait_vblank)

#define DRM_IOCTL_CRTC_GET_SEQUENCE	DRM_IOWR(0x3b, struct drm_crtc_get_sequence)
#define DRM_IOCTL_CRTC_QUEUE_SEQUENCE	DRM_IOWR(0x3c, struct drm_crtc_queue_sequence)

#define DRM_IOCTL_UPDATE_DRAW		DRM_IOW(0x3f, struct drm_update_draw)

#define DRM_IOCTL_MODE_GETRESOURCES	DRM_IOWR(0xA0, struct drm_mode_card_res)
#define DRM_IOCTL_MODE_GETCRTC		DRM_IOWR(0xA1, struct drm_mode_crtc)
#define DRM_IOCTL_MODE_SETCRTC		DRM_IOWR(0xA2, struct drm_mode_crtc)
#define DRM_IOCTL_MODE_CURSOR		DRM_IOWR(0xA3, struct drm_mode_cursor)
#define DRM_IOCTL_MODE_GETGAMMA		DRM_IOWR(0xA4, struct drm_mode_crtc_lut)
#define DRM_IOCTL_MODE_SETGAMMA		DRM_IOWR(0xA5, struct drm_mode_crtc_lut)
#define DRM_IOCTL_MODE_GETENCODER	DRM_IOWR(0xA6, struct drm_mode_get_encoder)
#define DRM_IOCTL_MODE_GETCONNECTOR	DRM_IOWR(0xA7, struct drm_mode_get_connector)
#define DRM_IOCTL_MODE_ATTACHMODE	DRM_IOWR(0xA8, struct drm_mode_mode_cmd) /* deprecated (never worked) */
#define DRM_IOCTL_MODE_DETACHMODE	DRM_IOWR(0xA9, struct drm_mode_mode_cmd) /* deprecated (never worked) */

#define DRM_IOCTL_MODE_GETPROPERTY	DRM_IOWR(0xAA, struct drm_mode_get_property)
#define DRM_IOCTL_MODE_SETPROPERTY	DRM_IOWR(0xAB, struct drm_mode_connector_set_property)
#define DRM_IOCTL_MODE_GETPROPBLOB	DRM_IOWR(0xAC, struct drm_mode_get_blob)
#define DRM_IOCTL_MODE_GETFB		DRM_IOWR(0xAD, struct drm_mode_fb_cmd)
#define DRM_IOCTL_MODE_ADDFB		DRM_IOWR(0xAE, struct drm_mode_fb_cmd)
#define DRM_IOCTL_MODE_RMFB		DRM_IOWR(0xAF, unsigned int)
#define DRM_IOCTL_MODE_PAGE_FLIP	DRM_IOWR(0xB0, struct drm_mode_crtc_page_flip)
#define DRM_IOCTL_MODE_DIRTYFB		DRM_IOWR(0xB1, struct drm_mode_fb_dirty_cmd)

#define DRM_IOCTL_MODE_CREATE_DUMB DRM_IOWR(0xB2, struct drm_mode_create_dumb)
#define DRM_IOCTL_MODE_MAP_DUMB    DRM_IOWR(0xB3, struct drm_mode_map_dumb)
#define DRM_IOCTL_MODE_DESTROY_DUMB    DRM_IOWR(0xB4, struct drm_mode_destroy_dumb)
#define DRM_IOCTL_MODE_GETPLANERESOURCES DRM_IOWR(0xB5, struct drm_mode_get_plane_res)
#define DRM_IOCTL_MODE_GETPLANE	DRM_IOWR(0xB6, struct drm_mode_get_plane)
#define DRM_IOCTL_MODE_SETPLANE	DRM_IOWR(0xB7, struct drm_mode_set_plane)
#define DRM_IOCTL_MODE_ADDFB2		DRM_IOWR(0xB8, struct drm_mode_fb_cmd2)
#define DRM_IOCTL_MODE_OBJ_GETPROPERTIES	DRM_IOWR(0xB9, struct drm_mode_obj_get_properties)
#define DRM_IOCTL_MODE_OBJ_SETPROPERTY	DRM_IOWR(0xBA, struct drm_mode_obj_set_property)
#define DRM_IOCTL_MODE_CURSOR2		DRM_IOWR(0xBB, struct drm_mode_cursor2)
#define DRM_IOCTL_MODE_ATOMIC		DRM_IOWR(0xBC, struct drm_mode_atomic)
#define DRM_IOCTL_MODE_CREATEPROPBLOB	DRM_IOWR(0xBD, struct drm_mode_create_blob)
#define DRM_IOCTL_MODE_DESTROYPROPBLOB	DRM_IOWR(0xBE, struct drm_mode_destroy_blob)

#define DRM_IOCTL_SYNCOBJ_CREATE	DRM_IOWR(0xBF, struct drm_syncobj_create)
#define DRM_IOCTL_SYNCOBJ_DESTROY	DRM_IOWR(0xC0, struct drm_syncobj_destroy)
#define DRM_IOCTL_SYNCOBJ_HANDLE_TO_FD	DRM_IOWR(0xC1, struct drm_syncobj_handle)
#define DRM_IOCTL_SYNCOBJ_FD_TO_HANDLE	DRM_IOWR(0xC2, struct drm_syncobj_handle)
#define DRM_IOCTL_SYNCOBJ_WAIT		DRM_IOWR(0xC3, struct drm_syncobj_wait)
#define DRM_IOCTL_SYNCOBJ_RESET		DRM_IOWR(0xC4, struct drm_syncobj_array)
#define DRM_IOCTL_SYNCOBJ_SIGNAL	DRM_IOWR(0xC5, struct drm_syncobj_array)

#define DRM_IOCTL_MODE_CREATE_LEASE	DRM_IOWR(0xC6, struct drm_mode_create_lease)
#define DRM_IOCTL_MODE_LIST_LESSEES	DRM_IOWR(0xC7, struct drm_mode_list_lessees)
#define DRM_IOCTL_MODE_GET_LEASE	DRM_IOWR(0xC8, struct drm_mode_get_lease)
#define DRM_IOCTL_MODE_REVOKE_LEASE	DRM_IOWR(0xC9, struct drm_mode_revoke_lease)

#define DRM_IOCTL_SYNCOBJ_TIMELINE_WAIT	DRM_IOWR(0xCA, struct drm_syncobj_timeline_wait)
#define DRM_IOCTL_SYNCOBJ_QUERY		DRM_IOWR(0xCB, struct drm_syncobj_timeline_array)
#define DRM_IOCTL_SYNCOBJ_TRANSFER	DRM_IOWR(0xCC, struct drm_syncobj_transfer)
#define DRM_IOCTL_SYNCOBJ_TIMELINE_SIGNAL	DRM_IOWR(0xCD, struct drm_syncobj_timeline_array)

#define DRM_IOCTL_MODE_GETFB2		DRM_IOWR(0xCE, struct drm_mode_fb_cmd2)
```

phytium_display_drm_driver定义
```c
// drivers/gpu/drm/phytium/phytium_display_drv.c
struct drm_driver phytium_display_drm_driver = {
    .driver_features            = DRIVER_HAVE_IRQ |
                                  DRIVER_MODESET |
                                  DRIVER_ATOMIC |
                                  DRIVER_GEM,
    .load                       = phytium_display_load,
    .unload                     = phytium_display_unload,
    .lastclose                  = drm_fb_helper_lastclose,
    .irq_handler                = phytium_display_irq_handler,
    .irq_preinstall             = phytium_irq_preinstall,
    .irq_uninstall              = phytium_irq_uninstall,
    .prime_handle_to_fd         = drm_gem_prime_handle_to_irq,
    .prime_fd_to_handle         = drm_gem_prime_fd_to_handle,
    .gem_prime_export           = drm_gem_prime_export,
    .gem_prime_import           = drm_gem_prime_import,
    .gem_prime_import_sg_table  = phytium_gem_prime_import_sg_table,
    .gem_prime_mmap             = phytium_gem_prime_mmap,
    .dump_create                = phytium_gem_dumb_create,
    .dump_destroy               = phytium_gem_dumb_destroy,
    .ioctls                     = phytium_ioctls,
    .num_ioctls                 = ARRAY_SIZE(phytium_ioctls),
    .fops                       = &phytium_drm_driver_fops,
    .name                       = DRV_NAME,
    .desc                       = DRV_DESC,
    .date                       = DRV_DATE,
    .major                      = DRV_MAJOR,
    .minor                      = DRV_MINOR,
};
```

```c
// drivers/gpu/drm/phytium/phytium_platform.c
static int phytium_platform_probe(struct platform_device *pdev)
{
	struct phytium_display_private *priv = NULL;
	struct drm_device *dev = NULL;
	int ret = 0;

    // 分配一个drm_device，并进行一些基础的初始化
	dev = drm_dev_alloc(&phytium_display_drm_driver, &pdev->dev);
	if (IS_ERR(dev)) {
		DRM_ERROR("failed to allocate drm_device\n");
		return PTR_ERR(dev);
	}

	dev_set_drvdata(&pdev->dev, dev);
	// 只能访问40bit dma地址
	dma_set_mask(&pdev->dev, DMA_BIT_MASK(40));

	// phytium_display_private结构体初始化
	priv = phytium_platform_private_init(pdev);
	if (priv)
		dev->dev_private = priv;
	else
		goto failed_platform_private_init;

	ret = phytium_platform_carveout_mem_init(pdev, priv);
	if (ret) {
		DRM_ERROR("failed to init system carveout memory\n");
		goto failed_carveout_mem_init;
	}

	// 注册drm_device，主要的初始化过程在这里面
	ret = drm_dev_register(dev, 0);
	if (ret) {
		DRM_ERROR("failed to register drm dev\n");
		goto failed_register_drm;
	}

	phytium_dp_hpd_irq_setup(dev, true);

	return 0;

failed_register_drm:
	phytium_platform_carveout_mem_fini(pdev, priv);
failed_carveout_mem_init:
	phytium_platform_private_fini(pdev);
failed_platform_private_init:
	dev_set_drvdata(&pdev->dev, NULL);
	drm_dev_put(dev);
	return -1;
}
```

drm_dev_register()分析
```c
int drm_dev_register(struct drm_device *dev, unsigned long flags)
{
	struct drm_driver *driver = dev->driver;
	int ret;

	if (!driver->load)
		drm_mode_config_validate(dev);

	WARN_ON(!dev->managed.final_kfree);

	if (drm_dev_needs_global_mutex(dev))
		mutex_lock(&drm_global_mutex);
	// phytium dc驱动没有DRM_RENDER特性
	ret = drm_minor_register(dev, DRM_MINOR_RENDER);
	if (ret)
		goto err_minors;
	// 创建sysfs目录，/sys/kernel/debug/dri/目录
	ret = drm_minor_register(dev, DRM_MINOR_PRIMARY);
	if (ret)
		goto err_minors;

	ret = create_compat_control_link(dev);
	if (ret)
		goto err_minors;

	dev->registered = true;

	if (dev->driver->load) {
		ret = dev->driver->load(dev, flags);
		if (ret)
			goto err_minors;
	}

	// register_all接口里面主要是各个对象funcs中late_register接口的调用
	// Phytium驱动中没有实现该接口
	if (drm_core_check_feature(dev, DRIVER_MODESET))
		drm_modeset_register_all(dev);

	ret = 0;

	DRM_INFO("Initialized %s %d.%d.%d %s for %s on minor %d\n",
		 driver->name, driver->major, driver->minor,
		 driver->patchlevel, driver->date,
		 dev->dev ? dev_name(dev->dev) : "virtual device",
		 dev->primary->index);

	goto out_unlock;

err_minors:
	remove_compat_control_link(dev);
	drm_minor_unregister(dev, DRM_MINOR_PRIMARY);
	drm_minor_unregister(dev, DRM_MINOR_RENDER);
out_unlock:
	if (drm_dev_needs_global_mutex(dev))
		mutex_unlock(&drm_global_mutex);
	return ret;
}
EXPORT_SYMBOL(drm_dev_register);
```

phytium_display_load()分析
```c
static int phytium_display_load(struct drm_device *dev, unsigned long flags)
{
	struct phytium_display_private *priv = dev->dev_private;
	int ret = 0;
	
	// 初始化drm_device->vblank，创建两个工作线程card0-crtc0 card0-crtc1
	ret = drm_vblank_init(dev, priv->info.num_pipes);
	if (ret) {
		DRM_ERROR("vblank init failed\n");
		goto failed_vblank_init;
	}

	// crtc和plane初始化
	ret = phytium_modeset_init(dev);
	if (ret) {
		DRM_ERROR("phytium_modeset_init failed\n");
		goto failed_modeset_init;
	}

	if (priv->support_memory_type & MEMORY_TYPE_VRAM)
		priv->vram_hw_init(priv);

	ret = drm_irq_install(dev, priv->irq);
	if (ret) {
		DRM_ERROR("install irq failed\n");
		goto failed_irq_install;
	}

	ret = phytium_drm_fbdev_init(dev);
	if (ret)
		DRM_ERROR("failed to init dev\n");

	phytium_debugfs_display_register(priv);

	return ret;

failed_irq_install:
	drm_mode_config_cleanup(dev);
failed_modeset_init:
failed_vblank_init:
	return ret;
}
```

phytium_crtc_init()分析
```c
int phytium_crtc_init(struct drm_device *dev, int phys_pipe)
{
	struct phytium_crtc *phytium_crtc;
	struct phytium_crtc_state *phytium_crtc_state;
	struct phytium_plane *phytium_primary_plane = NULL;
	struct phytium_plane *phytium_cursor_plane = NULL;
	struct phytium_display_private *priv = dev->dev_private;
	int ret;

	phytium_crtc = kzalloc(sizeof(*phytium_crtc), GFP_KERNEL);
	if (!phytium_crtc) {
		ret = -ENOMEM;
		goto failed_malloc_crtc;
	}

	phytium_crtc_state = kzalloc(sizeof(*phytium_crtc_state), GFP_KERNEL);
	if (!phytium_crtc_state) {
		ret = -ENOMEM;
		goto failed_malloc_crtc_state;
	}

	phytium_crtc_state->base.crtc = &phytium_crtc->base;
	phytium_crtc->base.state = &phytium_crtc_state->base;
	// phys_pipe代表的应该是两路display
	phytium_crtc->phys_pipe = phys_pipe;

	// 在phytium_crtc中对硬件相关的结构进行了剥离
	if (IS_PX210(priv)) {
		// x100的crtc hw相关配置drivers/gpu/drm/phytium/px210_dc.c中
		phytium_crtc->dc_hw_config_pix_clock = px210_dc_hw_config_pix_clock;
		phytium_crtc->dc_hw_disable = px210_dc_hw_disable;
		phytium_crtc->dc_hw_reset = NULL;
		priv->dc_reg_base[phys_pipe] = PX210_DC_BASE(phys_pipe);
		priv->dcreq_reg_base[phys_pipe] = PX210_DCREQ_BASE(phys_pipe);
		priv->address_transform_base = PX210_ADDRESS_TRANSFORM_BASE;
	} else if (IS_PE220X(priv)) {
		// e2000的crtc hw相关配置drivers/gpu/drm/phytium/pe220x_dc.c中
		phytium_crtc->dc_hw_config_pix_clock = pe220x_dc_hw_config_pix_clock;
		phytium_crtc->dc_hw_disable = pe220x_dc_hw_disable;
		phytium_crtc->dc_hw_reset = pe220x_dc_hw_reset;
		priv->dc_reg_base[phys_pipe] = PE220X_DC_BASE(phys_pipe);
		priv->dcreq_reg_base[phys_pipe] = 0x0;
		priv->address_transform_base = PE220X_ADDRESS_TRANSFORM_BASE;
	}

	// 每个DC有两个plane，分别是主图层和鼠标图层
	phytium_primary_plane = phytium_primary_plane_create(dev, phys_pipe);
	if (IS_ERR(phytium_primary_plane)) {
		ret = PTR_ERR(phytium_primary_plane);
		DRM_ERROR("create primary plane failed, phys_pipe(%d)\n", phys_pipe);
		goto failed_create_primary;
	}

	phytium_cursor_plane = phytium_cursor_plane_create(dev, phys_pipe);
	if (IS_ERR(phytium_cursor_plane)) {
		ret = PTR_ERR(phytium_cursor_plane);
		DRM_ERROR("create cursor plane failed, phys_pipe(%d)\n", phys_pipe);
		goto failed_create_cursor;
	}

	// crtc初始化
	ret = drm_crtc_init_with_planes(dev, &phytium_crtc->base,
					&phytium_primary_plane->base,
					&phytium_cursor_plane->base,
					&phytium_crtc_funcs,
					"phys_pipe %d", phys_pipe);

	if (ret) {
		DRM_ERROR("init crtc with plane failed, phys_pipe(%d)\n", phys_pipe);
		goto failed_crtc_init;
	}
	drm_crtc_helper_add(&phytium_crtc->base, &phytium_crtc_helper_funcs);
	drm_crtc_vblank_reset(&phytium_crtc->base);
	// 配置gamma校正
	drm_mode_crtc_set_gamma_size(&phytium_crtc->base, GAMMA_INDEX_MAX);
	drm_crtc_enable_color_mgmt(&phytium_crtc->base, 0, false, GAMMA_INDEX_MAX);
	if (phytium_crtc->dc_hw_reset)
		phytium_crtc->dc_hw_reset(&phytium_crtc->base);
	phytium_crtc_gamma_init(&phytium_crtc->base);

	return 0;

failed_crtc_init:
failed_create_cursor:
	/* drm_mode_config_cleanup() will free any crtcs/planes already initialized */
failed_create_primary:
	kfree(phytium_crtc_state);
failed_malloc_crtc_state:
	kfree(phytium_crtc);
failed_malloc_crtc:
	return ret;
}
```

phytium_dp_init()分析
```c
int phytium_dp_init(struct drm_device *dev, int port)
{
	struct phytium_display_private *priv = dev->dev_private;
	struct phytium_dp_device *phytium_dp = NULL;
	int ret, type;

	DRM_DEBUG_KMS("%s: port %d\n", __func__, port);
	phytium_dp = kzalloc(sizeof(*phytium_dp), GFP_KERNEL);
	if (!phytium_dp) {
		ret = -ENOMEM;
		goto failed_malloc_dp;
	}

	phytium_dp->dev = dev;
	phytium_dp->port = port;

	if (IS_PX210(priv)) {
		px210_dp_func_register(phytium_dp);
		priv->dp_reg_base[port] = PX210_DP_BASE(port);
		priv->phy_access_base[port] = PX210_PHY_ACCESS_BASE(port);
	} else if (IS_PE220X(priv)) {
		pe220x_dp_func_register(phytium_dp);
		priv->dp_reg_base[port] = PE220X_DP_BASE(port);
		priv->phy_access_base[port] = PE220X_PHY_ACCESS_BASE(port);
	}

	// 根据设备树中配置的ddp_mask属性来判断
	if (phytium_dp_is_edp(phytium_dp, port)) {
		phytium_dp->is_edp = true;
		type = DRM_MODE_CONNECTOR_eDP;
		phytium_dp_panel_init_backlight_funcs(phytium_dp);
		phytium_edp_backlight_off(phytium_dp);
		phytium_edp_panel_poweroff(phytium_dp);
	} else {
		phytium_dp->is_edp = false;
		type = DRM_MODE_CONNECTOR_DisplayPort;
	}
	// DP复位和一些初始化工作，完善phtium_dp结构体
	ret = phytium_dp_hw_init(phytium_dp);
	if (ret) {
		DRM_ERROR("failed to initialize dp %d\n", phytium_dp->port);
		goto failed_init_dp;
	}
	// 初始phytium_dp->encoder的某些成员，将其加入到drm_device->mode_config.encoder_list上
	ret = drm_encoder_init(dev, &phytium_dp->encoder,
			       &phytium_encoder_funcs,
			       DRM_MODE_ENCODER_TMDS, "DP %d", port);
	if (ret) {
		DRM_ERROR("failed to initialize encoder with drm\n");
		goto failed_encoder_init;
	}
	drm_encoder_helper_add(&phytium_dp->encoder, &phytium_encoder_helper_funcs);
	phytium_dp->encoder.possible_crtcs = phytium_get_encoder_crtc_mask(phytium_dp, port);

	phytium_dp->connector.dpms   = DRM_MODE_DPMS_OFF;
	phytium_dp->connector.polled = DRM_CONNECTOR_POLL_CONNECT | DRM_CONNECTOR_POLL_DISCONNECT;
	// 初始化connector，对于E2000Q DEMO板分别是DP-1 DP-2
	ret = drm_connector_init(dev, &phytium_dp->connector, &phytium_connector_funcs,
				 type);
	if (ret) {
		DRM_ERROR("failed to initialize connector with drm\n");
		goto failed_connector_init;
	}
	drm_connector_helper_add(&phytium_dp->connector, &phytium_connector_helper_funcs);
	drm_connector_attach_encoder(&phytium_dp->connector, &phytium_dp->encoder);

	ret = phytium_dp_audio_codec_init(phytium_dp, port);
	if (ret) {
		DRM_ERROR("failed to initialize audio codec\n");
		goto failed_connector_init;
	}

	phytium_dp->train_retry_count = 0;
	INIT_WORK(&phytium_dp->train_retry_work, phytium_dp_train_retry_work_fn);
	drm_connector_register(&phytium_dp->connector);

	return 0;
failed_connector_init:
failed_encoder_init:
failed_init_dp:
	kfree(phytium_dp);
failed_malloc_dp:
	return ret;
}
```

phytium_primary_plane_create()分析
```c
// drivers/gpu/drm/phytium/pe220x_dc.c
struct phytium_plane *phytium_primary_plane_create(struct drm_device *dev, int phys_pipe)
{
	struct phytium_display_private *priv = dev->dev_private;
	struct phytium_plane *phytium_plane = NULL;
	struct phytium_plane_state *phytium_plane_state = NULL;
	int ret = 0;
	unsigned int flags = 0;
	const uint32_t *formats = NULL;
	uint32_t format_count;
	const uint64_t *format_modifiers;

	phytium_plane = kzalloc(sizeof(*phytium_plane), GFP_KERNEL);
	if (!phytium_plane) {
		ret = -ENOMEM;
		goto failed_malloc_plane;
	}

	phytium_plane_state = kzalloc(sizeof(*phytium_plane_state), GFP_KERNEL);
	if (!phytium_plane_state) {
		ret = -ENOMEM;
		goto failed_malloc_plane_state;
	}
	phytium_plane_state->base.plane = &phytium_plane->base;
	phytium_plane_state->base.rotation = DRM_MODE_ROTATE_0;
	phytium_plane->base.state = &phytium_plane_state->base;
	phytium_plane->phys_pipe = phys_pipe;

	if (IS_PX210(priv)) {
		phytium_plane->dc_hw_plane_get_format = px210_dc_hw_plane_get_primary_format;
		phytium_plane->dc_hw_update_dcreq = px210_dc_hw_update_dcreq;
		phytium_plane->dc_hw_update_primary_hi_addr = px210_dc_hw_update_primary_hi_addr;
		phytium_plane->dc_hw_update_cursor_hi_addr = NULL;
	}  else if (IS_PE220X(priv)) {
		phytium_plane->dc_hw_plane_get_format = pe220x_dc_hw_plane_get_primary_format;
		phytium_plane->dc_hw_update_dcreq = NULL;
		phytium_plane->dc_hw_update_primary_hi_addr = pe220x_dc_hw_update_primary_hi_addr;
		phytium_plane->dc_hw_update_cursor_hi_addr = NULL;
	}

	// 获取像素颜色格式
	phytium_plane->dc_hw_plane_get_format(&format_modifiers, &formats, &format_count);
	// 创建plane
	ret = drm_universal_plane_init(dev, &phytium_plane->base, 0x0,
				       &phytium_plane_funcs, formats,
				       format_count,
				       format_modifiers,
				       DRM_PLANE_TYPE_PRIMARY, "primary %d", phys_pipe);

	if (ret)
		goto failed_plane_init;

	flags = DRM_MODE_ROTATE_0;
	drm_plane_create_rotation_property(&phytium_plane->base, DRM_MODE_ROTATE_0, flags);
	drm_plane_helper_add(&phytium_plane->base, &phytium_plane_helper_funcs);

	return phytium_plane;
failed_plane_init:
	kfree(phytium_plane_state);
failed_malloc_plane_state:
	kfree(phytium_plane);
failed_malloc_plane:
	return ERR_PTR(ret);
}
```

### phytium drm驱动内核参数

phytium drm驱动内核参数的路径`/sys/module/phytium_dc_drm/parameters`，内核参数如下：

1. dc_fake_mode_enable:
2. dc_fast_training_check：
3. num_source_rates： 设置最大的dp lane速率，默认为8.1Gbps
4. source_max_lane_count： 设置dp lane数量，默认为4
5. link_dynamic_adjust： 是否根据显示模式动态调整训练参数，默认为是

## 4. Phytium D3000M DRM驱动

### DC控制器参数

DC是显示控制器，主要完成将CPU、GPU、VPU处理后的图像数据，按照Display协议处理后送给DP Phy接入显示器进行显示。DC的IP来自于vivante，是全新的IP，DC控制器的参数如下：

+ D3000M的DC模块集成2个IP，其中DC0控制1路显示，DC1控制2路显示
+ 支持3路独立显示，每路能力达到3840x2160@60fps，如果两路独立输出，两路均支持3840x2140@60fps，如果内屏加外扩的两路，则外扩两路支持3840x2160@30fps
+ 支持视频叠加（Alpha Blending），包含1个Video，2个Overlay和1个Cursor图层
+ DP/eDP最高支持8.1Gbps链路速率，eDP速率支持0.27Gbps的倍数，支持eDP高级电源管理（ALPM）和面板自刷新（PSR/PSR2）

### DP参数

+ 兼容DisplayPort 1.4 / Embedded DisplayPort 1.4a协议
+ 支持音频数据通道
+ 最高支持每个颜色通道16 bit深度
+ 支持RGB、YUV像素格式输入
+ 支持1、2、4 lane模式
+ DP、eDP最高支持8.1Gbps链路速率，eDP速率支持0.27Gbps的倍数
+ **支持面板自刷新 PSR/PSR2**
+ 支持热插拔

DP在之前E2000的DP Phy的基础上增加了VRR功能（可变刷新率，Variable Refresh Rate）和面板自刷新技术（Panel Self Refresh，PSR），用于优化显示功耗

### D3000M DC设备树节点

```c
dc8200@26ca0000 {
	compatible = "phytium,dc-1.0";
	reg = <0x00 0x26ca0000 0x00 0x1e000>,
		  <0x22 0x00 0x00 0x20000000>,
		  <0x00 0x26fcc080 0x00 0x100>;
	interrupts = <0x00 0x37 0x04>, <0x00 0x3a 0x04>, <0x00 0x41 0x04>, <0x00 0x42 0x04>, <0x00 0x43 0x04>, <0x00 0x6a 0x04>, <0x00 0x6b 0x04>, <0x00 0x6c 0x04>;
	pipe_mask = [07];
	phy_mode = <0x01 0x01 0x01>;
	edp_mask = [01];
	water_mark = <0x5666 0x5666 0x5666>;
	qos = <0xf0 0xf0>;
	overlay_enable = [01];
	#address-cells = <0x01>;
	#size-cells = <0x00>;
};
```

1. 设备树reg一共描述3段地址：
   + 第一段： DC控制器寄存器地址，基地址为 0x26CA0000，长度为 0x1E000
   + 第二段： 显存地址，起始地址为 0x2200000000，显存大小为 0x20000000，即512MB
   + 第三段： 安全态寄存器地址，基地址为 0x26FCC090，长度为 0x100
2. 设备树中断部分一共描述了8个中断，分别为：
	+ DC0 控制器中断（vblank中断）
	+ DC1 控制器中断 (vblank中断)
	+ DP0 热插拔中断
	+ DP1 热插拔中断
	+ DP2 热插拔中断
	+ DP0 热插拔上电中断
	+ DP1 热插拔上电中断
	+ DP2 热插拔上电中断
3. phy_mode参数表示DP使用到的lane情况，主要和其他外设的复用有关
4. water_mark：通过设定“水位线”，帮助显示控制器何时从内存读取数据，优化内存带宽使用

### 驱动结构

D3000M的整个DRM显示驱动由vendor提供的DC硬件部分的代码（文件夹8x00下的代码）与以前Phytium DRM驱动中的DP驱动代码结合而来

### 相关结构体

#### `struct phy_drm_private`

`struct phy_drm_private`，D3000M DRM驱动的私有结构体，`drm_device->dev_private`指向该结构体

```c
// phytium/phy_drv.h
struct phy_drm_private {
	/* 指向linux kernel device */
	struct device *dma_dev;
	/* when we have more than one display core, this need to be an array */
	struct device *dc_dev;

	struct iommu_domain *domain;
#ifdef CONFIG_PHYTIUM_MMU
	dc_mmu * mmu;
#endif

#ifdef CONFIG_PHYTIUM_DEBUG
	struct file *dc_capture_fp;
#endif

	/* 对齐相关参数 */
	unsigned int pitch_alignment;
	unsigned int addr_alignment;

	u8 intr_dest;
	u32 intr_mask;

	struct phytium_device_info info;
	void __iomem *regs;
	/* 存放安全寄存器基地址映射后的地址 */
	void __iomem *se_regs;
	uint32_t dp_reg_base[DISPLAY_NUM];
	uint32_t dplp_reg_base[DISPLAY_NUM];
	uint32_t address_transform_base;
	uint32_t phy_access_base[DISPLAY_NUM];

#ifdef CONFIG_PHYTIUM_PCIE
	struct pci_dev *pdev;
#else
	/* 指向platform device */
	struct platform_device *pdev;
#endif
	/* 指向drm_device，drm_device->dev_private指向phy_drm_private */
	struct drm_device *drm_dev;

	struct gen_pool *mem_pool;

	/* 驱动模块的start_address参数也可以手动指定显存的地址 */
	/* 显存起始物理地址 */
	unsigned long mem_pool_start_address_phy;
	/* 存放显存映射后的虚拟地址 */
	void *mem_pool_start_address_virt;
	/* 显存大小 */
	unsigned long  mem_pool_size;

	/* fb_dev */
	struct drm_fb_helper fbdev_helper;
	struct phy_gem_object *fbdev_phytium_gem;
	struct work_struct fbdev_init_work;
	bool phytium_log_enable;

	/* low_power_enable 默认是打开的 */
	bool low_power_enable[DISPLAY_NUM];

	/*edp_pwm*/
	uint32_t edp_pwm_base;

	/*dp_hotplug_mutex*/
	struct mutex power_hpd_mutex;
	struct list_head gem_list_head;
};

```

#### `struct phy_dc`

```c
// phytium/8x00/phy_dc.h

struct phy_dc {
	struct phy_crtc *crtc[DC_DISPLAY_NUM];
	struct dc_hw hw;
#ifdef CONFIG_PHYTIUM_DEC
	struct dc_dec400l dec400l;
#endif

	bool first_frame;

	struct phy_dc_plane planes[PLANE_NUM];

	struct simple_encoder *encoder[DC_OUTPUT_NUM];

#ifdef CONFIG_PHYTIUM_WRITEBACK
	struct phy_writeback_connector *writeback[DC_WB_NUM];
#endif

#ifdef CONFIG_PHYTIUM_VIRTUAL_DISPLAY
	struct phy_virtual_display *vd[DC_OUTPUT_NUM];
#endif
};

// phytium/8x00/phy_dc_hw.h
struct dc_hw {
	enum dc_chip_rev rev;
	/* 存放DC控制器寄存器基地址映射后的地址 */
	void *hi_base;
	/* 同hi_base */
	void *reg_base;
#if defined(CONFIG_PHYTIUM_MMU) || defined(CONFIG_PHYTIUM_WRITEBACK)
	void *sec_base;
#endif

#ifdef CONFIG_PHYTIUM_DEBUG
	struct file *dc_capture_fp;
#endif
	struct dc_hw_display display[DC_DISPLAY_NUM];
	struct dc_hw_plane plane[DC_LAYER_NUM];
	struct dc_hw_cursor cursor[DC_CURSOR_NUM];
	struct dc_hw_wb wb[DC_WB_NUM];
	struct dc_hw_qos qos;
	struct dc_hw_funcs *func;
	struct dc_hw_sub_funcs *sub_func;
	/* 存储DC控制器的一些参数信息，这些信息是代码里面固定配置好的 */
	struct phy_dc_info *info;
	/* 存储视频输出信息，也是代码里面固定配置好的 */
	const struct phy_output_info *output_info;
	int total_pipes;
	int pipe_mask;
	struct drm_device *drm_dev;
	bool phytium_log_enable;
	unsigned char overlay_enable;
};

// 开启两路DC的参数信息
static struct phy_dc_info phytium_2_dc_info = {
	.name = "DC8200",
	.plane_num = ARRAY_SIZE(phytium_2_dc_hw_planes),
	.planes = phytium_2_dc_hw_planes,
	.layer_num = 6,
	.display_num = ARRAY_SIZE(phytium_2_dc_hw_displays),
	.displays = phytium_2_dc_hw_displays,
	.output_num = ARRAY_SIZE(phytium_2_dc_output_info),
	.wb_num = ARRAY_SIZE(phytium_2_dc_hw_wbs),
	.write_back = phytium_2_dc_hw_wbs,
	.max_bpc = 10,
	.pitch_alignment = 128,
	.addr_alignment = 256,
	.max_blend_layer = 6,
	.max_gamma_size = GAMMA_SIZE,
	.gamma_bits = 12,
	.std_color_lut = true,
	.pipe_sync = false,
	.mmu_prefetch = false,
	.panel_sync = false,
	.cap_dec = true,
};
```

#### 驱动probe过程

![](https://raw.githubusercontent.com/JackHuang021/images/master/D3000M+DRM.png)

## 4. libdrm源码编译

libdrm 源码下载地址：[https://dri.freedesktop.org/libdrm/](https://dri.freedesktop.org/libdrm/)

## 4. DRM测试

### 4.1 使用libdrm测试显示流程


### 4.x 显示问题调试

针对xorg的显示问题排查步骤

1. 查看xorg log，`cat /var/log/Xorg.0.log`，从log信息中查找(WW)和(EE)相关log

2. 查看drm sysfs目录信息`/sys/class/drm`，

3. 打开drm驱动调试信息，增加内核启动参数`drm.debug`，调试级别如下：

	```bash
	root@Ubuntu:~# modinfo -p drm
	name:           drm
	vblankoffdelay:Delay until vblank irq auto-disable [msecs] (0: never disable, <0: disable immediately) (int)
	timestamp_precision_usec:Max. error on timestamps [usecs] (int)
	debug:Enable debug output, where each bit enables a debug category.
			Bit 0 (0x01)  will enable CORE messages (drm core code)
			Bit 1 (0x02)  will enable DRIVER messages (drm controller code)
			Bit 2 (0x04)  will enable KMS messages (modesetting code)
			Bit 3 (0x08)  will enable PRIME messages (prime code)
			Bit 4 (0x10)  will enable ATOMIC messages (atomic code)
			Bit 5 (0x20)  will enable VBL messages (vblank code)
			Bit 7 (0x80)  will enable LEASE messages (leasing code)
			Bit 8 (0x100) will enable DP messages (displayport code) (int)
	edid_fixup:Minimum number of valid EDID header bytes (0-8, default 6) (int)
	```

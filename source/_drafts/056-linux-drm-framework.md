---
title: Linux DRM显示框架
date: 2023-09-26 09:21:23
tags:
  - Linux
  - drm
categories: Linux
---

### DRM框架简述
DRM(Direct Rendering Manager)，DRM将现代显示领域中会涉及的一些操作进行分层并使这些模块独立，如果上层应用想操作显存、显示效果抑或是GPU，都必须在一些框架的约束下进行

可以从用户空间、内核空间的两个角度去了解DRM框架
+ 用户空间(libdrm driver)：libdrm，drm框架在用户空间的lib
+ 内核空间(drm driver)：
    + KMS(Kernel Mode Settings，内核显示模式设置)
    + GEM(Graphic Execution Manager，图形执行管理器)

DRM的发展历史: [https://blog.csdn.net/hexiaolong2009/article/details/88075520](https://blog.csdn.net/hexiaolong2009/article/details/88075520)

#### libdrm
DRM框架在用户空间提供的Lib，用户或应用程序在用户空间调用libdrm提供的库函数， 即可访问到显示的资源，并对显示资源进行管理和使用。这样通过libdrm对显示资源进行统一访问，libdrm将命令传递到内核最终由DRM驱动接管各应用的请求并处理， 可以有效避免访问冲突。

#### KMS（Kernel Mode Settings）
KMS主要负责两个功能：显示参数、显示控制，这两个基本功能是显示驱动必须具备的能力，在DRM框架下，为了将这两部分适配符合现代显示设备逻辑，又分出了几部分子模块配合框架(CRTC, ENCODER, CONNECTOR, PLANE, FB, VBLANK, property)

##### Plane
基本的显示控制单位，每个图像拥有一个Plane，Plane的属性控制着图像的显示区域、图像翻转、色彩混合方式等，最终图像经过Plane并通过CRTC组件，得到多个图像的混合显示或单独显示的功能

##### CRTC
用于控制显卡输出信号，将帧缓存中的图像数据按照一定的方式输出到显示器上，并控制显示器的显示模式、分辨率、刷新率等参数，在DRM中有多个显存，可以通过CRTC来控制要显示的那个显存

##### Encoder
用于控制将CRTC输出的图像信号转换成一定格式的数字信号，通常用于连接显示器等显示设备，每个CRTC可以有一个或者多个Encoder

##### Connector
通常用于将Encoder输出信号传递给显示器，并与显示器建立连接，每个Encoder可以有一个或多个Connector

##### Plane
硬件图层，负责获取显存，再输出到CRTC里，可以看做是一个显示器的图层，每个CRTC中至少要有一个Plane，通常会有多个Plane，每个Plane可以分别设置自己的属性，从而实现多个图像内容的叠加

##### FrameBuffer
帧缓存，用于存储屏幕上的每个像素点的颜色信息，只用于描述显存信息（如format、pitch、size等），不负责显存的分配释放

##### VBLANK
软件和硬件的同步机制，RGB时序中的垂直消影区，软件通常使用硬件VSYNC来实现

##### property
原子操作的基础，任何你想设置的参数，都可以做成property，供用户空间使用，是DRM驱动中最灵活、最方便的Mode setting机制

#### GEM（Generic DRM Memory Management）
GEM负责对DRM使用的内存进行管理，GEM框架提供的功能包括：内存分配和释放、命令执行、执行命令时的管理


![](https://raw.githubusercontent.com/JackHuang021/images/master/20231027140940.png)

#### dumb buffer
dumb buffer代表所有的绘图操作都是由CPU来完成的framebuffer，它只是一种软件功能上的定义，与你系统上是否带GPU硬件无关。即使你的硬件支持GPU加速，也不妨碍你使用dumb buffer来做CPU纯软绘的工作。正因为dumb buffer的这一功能特性，使得它普遍应用于简单UI场景，如Android的Recovery模式。

#### prime
prime在DRM驱动中其实是一种buffer共享机制，他是基于dma-buf来实现的


### DRM驱动框架中常用的结构体

#### struct drm_mode_object
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
    void (*free_cb)(struct kref *kref);
};
```

#### struct drm_framebuffer
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

#### struct drm_plane
linux内核使用`struct drm_plane`表示一个plane，plane从一个drm_framebuffer接收输入数据，并将其传递给一个drm_crtc
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
	struct drm_plane *plane;

	/**
	 * @crtc:
	 *
	 * Currently bound CRTC, NULL if disabled. Do not write this directly,
	 * use drm_atomic_set_crtc_for_plane()
	 */
	struct drm_crtc *crtc;

	/**
	 * @fb:
	 *
	 * Currently bound framebuffer. Do not write this directly, use
	 * drm_atomic_set_fb_for_plane()
	 */
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


#### struct drm_crtc
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

#### struct drm_device
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

	/**
	 * @managed:
	 *
	 * Managed resources linked to the lifetime of this &drm_device as
	 * tracked by @ref.
	 */
	struct {
		/** @managed.resources: managed resources list */
		struct list_head resources;
		/** @managed.final_kfree: pointer for final kfree() call */
		void *final_kfree;
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

#### struct drm_mode_config
linux内核使用`struct drm_mode_config`来描述显示模式配置信息，`drm_mode_config`的主要功能之一是提供对显示器模式的管理和配置，包括添加、删除、修改、查询显示器模式的能力，在整个驱动的初始化过程中`struct drm_mode_config`会记录crtc plane等的信息。
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

#### struct drm_minor
`struct drm_minor`用来描述在/dev下的drm设备节点，drm core会根据`driver_features`来决定是否为`drm_device`中的primary, render, accel注册字符设备，同时在`/dev/dri`目录下创建相应的设备节点。
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
	struct drm_device *dev;
	/* debugfs 目录项 */
	struct dentry *debugfs_root;

	struct list_head debugfs_list;
	struct mutex debugfs_lock; /* Protects debugfs_list. */
};
```

#### struct drm_vblank_crtc
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
	struct drm_device *dev;
	/**
	 * @queue: Wait queue for vblank waiters.
	 */
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
	atomic64_t count;
	/**
	 * @time: Vblank timestamp corresponding to @count.
	 */
	ktime_t time;

	/**
	 * @refcount: Number of users/waiters of the vblank interrupt. Only when
	 * this refcount reaches 0 can the hardware interrupt be disabled using
	 * @disable_timer.
	 */
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

#### struct drm_gem_object
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

### phytium drm dc驱动

#### phytium E2000 DC控制器设备树描述
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

#### struct phytium_display_private结构体定义
```c
// drivers/gpu/drm/phytium/phytium_display_drv.h
struct phytium_display_private {
	/* hw */
	void __iomem *regs;
	void __iomem *vram_addr;
	struct phytium_device_info info;
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
	uint64_t mem_state[PHYTIUM_MEM_STATE_TYPE_COUNT];

	int dma_inited;
	struct dma_chan *dam_chan;
};
```

#### struct phytium_device_info结构体定义
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

#### struct phytium_dp_device结构体定义

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

#### phytium crtc相关结构体定义
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

#### phytium_plane结构体定义
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


#### 检测热插拔状态流程
DP检测热插拔状态由`phytium_dp_hw_get_hpd_state()`完成，通过读取寄存器`PHYTIUM_DP_INTERRUPT_RAW_STATUS 0x130`及`PHYTIUM_DP_SINK_HPD_STATE 0x128`来获取热插拔检测的状态，`struct phytium_dp_device`中的`struct phytium_dp_hpd_state`记录了热插拔的检测状态。



#### 热插拔中断过程
drm_driver->irq_handler()
	phytium_display_irq_handler()
		phytium_dp_hpd_irq_handler()
			phytium_dp_hpd_work_func()


phytium_dp_hpd_work_func()分析
```c

```

### DRM调试
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
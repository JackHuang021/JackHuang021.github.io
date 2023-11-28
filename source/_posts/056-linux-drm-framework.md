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
```c
struct drm_plane {
    struct drm_device *dev;
    struct list_head head;
    char *name;
    struct drm_modeset_lock mutex;
    struct drm_modeset_object base;

    uint32_t possible_crtcs;
    uint32_t *format_types;
    unsigned int format_count;

    bool format_default;
    uint64_t *modifiers;
    unsigned int modifier_count;

    struct drm_crtc *crtc;
    struct drm_framebuffer *fb;
    struct frm framebuffer *old_fb;

    const struct drm_plane_funcs *funcs;

    struct drm_object_properties properties;

    enum drm_plane_type type;
    unsigned index;

    const struct drm_plane_helper_funcs *helper_private;

    struct drm_plane_state *state;

    struct drm_property *alpha_property;
    struct drm_property *zpos_property;
    struct drm_property *rotation_property;
    struct drm_property *blend_mode_property;
    struct drm_property *color_encoding_property;
    struct drm_property *color_range_property;
};

// include/drm/drm_plane.h
struct drm_plane_funcs {
	// 为给定的CRTC和framebuffer启用并配置plane
	int (*update_plane)(struct drm_plane *plane,
					    struct drm_crtc, struct drm_framebuffer *fb,
						int crtc_x, int crtc_y,
						unsigned int crtc_w, unsigned int crtc_h,
						uint32_t src_x, uint32_t src_y,
						uint32_t src_w, uint32_t src_w,
						struct drm_modeset_acquire_ctx *ctx);

	// 关闭plane
	int (*disable_plane)(struct drm_plane *plane,
						 struct drm_modeset_acquire_ctx *ctx);

	// 清除plane所有的资源
	void (*destory)(struct drm_plane *plane);

	void (*reset)(struct drm_plane *plane);

	int (*set_property)(struct drm_plane *plane,
						struct drm_property *property, uint64_t val);

	struct drm_plane_state *(*atomic_duplicate_state)(struct drm_plane *plane);

	void (*atomic_destory_state)(struct drm_plane *plane,
								 struct drm_plane_state *state);

	int (*atomic_set_proprtty)(struct drm_plane *plane,
							   struct drm_plane_state *state,
							   struct drm_property *property,
							   uint64_t val);

	int (*atomic_get_property)(struct drm_plane *plane,
							   const struct drm_plane_state *state,
							   struct drm_property *property,
							   uint64_t *val);

	int (*late_register)(struct drm_plane *plane);

	void (*early_unregister)(struct drm_plane *plane);

	void (*atomic_print_state)(struct drm_printer *p,
							   const struct drm_plane_state *state);

	bool (*format_mod_support)(struct drm_plane *plane, uint32_t format,
							   uint64_t modifier);
};

// inlcude/drm/drm_modeset_helper_vtables.h
struct drm_plane_helper_funcs {
	int (*prepare_fb)(struct drm_plane *plane,
					  struct drm_plane_state *new_state);

	void (*cleanup_fb)(struct drm_plane,
					   struct drm_plane_state *old_state);

	int (*atomic_check)(struct drm_plane *plane,
						struct drm_plane_state *state);

	void (*atomic_update)(struct drm_plane *plane,
						  struct drm_plane_state *old_state);

	void (*atomic_disable)(struct drm_plane *plane,
						   struct drm_plane_state *old_state);

	int (*atomic_async_check)(struct drm_plane *plane,
							  struct drm_plane_state *state);

	void (*atomic_async_update)(struct drm_plane *plane,
								struct drm_plane_state *new_state);
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
```c
// include/drm/drm_device.h
struct drm_device {
    struct list_head legacy_dev_list;
    int if_version;
    struct kref ref;
    struct device *dev;

    struct {
        struct list_head resources;
        void *final_kfree;
        spinlock_t lock;
    }managed;

    struct drm_driver *driver;
    void *dev_private;
    struct drm_minor *primary;
    struct drm_minor *render;
    bool registered;
    struct drm_master *master;
    u32 driver_features;
    bool unplugged;
    struct inode *anon_inode;
    char *unique;
    struct mutex struct_mutex;
    struct mutex master_mutex;
    atomic_t open_count;
    struct mutex filelist_mutex;
    struct list_head filelist;
    struct list_head filelist_internal;
    struct mutex clientlist_mutex;
    struct list_head clientlist;

    bool irq_enabled;
    int irq;

    bool vblank_disable_immediate;
    struct drm_vblank_crtc *vblank;
    spinlock_t vblank_time_lock;
    spinlock_t vbl_lock;
    u32 max_vblank_count;
    struct list_head vblank_event_list;
    spinlock_t event_lock;
    struct drm_agp_head *agp;

    struct pci_dev *pdev;

    unsigned int num_crtcs;
    struct drm_mode_config mode_config;
    struct mutex object_name_lock;
    struct idr object_name_idr;
    struct drm_vma_offset_manager *vma_offset_manager;
    struct drm_vram_mm *vram_mm;
    enum switch_power_state switch_power_state;
    struct drm_fb_helper *fb_helper;
	...
};
```

#### struct drm_mode_config
```c
struct drm_mode_config {
	struct mutex mutex;
	struct drm_modeset_lock connection_mutex;
	struct drm_modeset_acquire_ctx *acquire_ctx;
	struct mutex idr_mutex;
	struct idr object_idr;
	struct idr tile_idr;

	struct mutex fb_lock;
	int num_fb;
	struct list_head fb_list;

	spinlock_t connector_list_lock;
	int num_connector;
	struct ida connector_ida;
	struct list_head connector_list;
	struct list_head connector_free_list;
	struct work_struct connector_free_work;

	int num_encoder;
	struct list_head encoder_list;

	int num_total_planel;
	struct list_head plane_list;

	int num_crtc;
	struct list_head crtc_list;

	struct list_head property_list;
	struct list_head privobj_list;

	int min_width;
	int min_height;
	int max_width;
	int max_height;
	const struct drm_mode_config_funcs *funcs;
	resource_size_t fb_base;

	bool poll_enabled;
	bool poll_running;
	bool delayed_event;
	struct delayed_work output_poll_work;

	struct mutex blob_lock;
	struct list_head property_blob_list;

	struct drm_property *edid_property;
	struct drm_property *dpms_property;
	struct drm_property *path_property;
	struct drm_property *tile_property;
	struct drm_property *link_status_property;
	struct drm_property *plane_type_property;
	struct drm_property *prop_src_x;
	struct drm_property *prop_src_y;
	struct drm_property *prop_src_w;
	struct drm_property *prop_src_h;
	struct drm_property *prop_crtc_x;
	struct drm_property *prop_crtc_y;
	struct drm_property *prop_crtc_w;
	struct drm_property *prop_crtc_h;
	struct drm_property *prop_fb_id;
	struct drm_property *prop_in_fence_id;
	struct drm_property *prop_out_fence_id;
	struct drm_property *prop_crtc_id;
	struct drm_property *prop_fb_damage_clips;
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
	 * @tv_mode_property: Optional TV property to select
	 * the output TV mode.
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
	 * @prefer_shadow_fbdev:
	 *
	 * Hint to framebuffer emulation to prefer shadow-fb rendering.
	 */
	bool prefer_shadow_fbdev;

	/**
	 * @fbdev_use_iomem:
	 *
	 * Set to true if framebuffer reside in iomem.
	 * When set to true memcpy_toio() is used when copying the framebuffer in
	 * drm_fb_helper.drm_fb_helper_dirty_blit_real().
	 *
	 * FIXME: This should be replaced with a per-mapping is_iomem
	 * flag (like ttm does), and then used everywhere in fbdev code.
	 */
	bool fbdev_use_iomem;

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
	 * @allow_fb_modifiers:
	 *
	 * Whether the driver supports fb modifiers in the ADDFB2.1 ioctl call.
	 */
	bool allow_fb_modifiers;

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

#### struct drm_driver
```c
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
	 * @irq_handler:
	 *
	 * Interrupt handler called when using drm_irq_install(). Not used by
	 * drivers which implement their own interrupt handling.
	 */
	irqreturn_t(*irq_handler) (int irq, void *arg);

	/**
	 * @irq_preinstall:
	 *
	 * Optional callback used by drm_irq_install() which is called before
	 * the interrupt handler is registered. This should be used to clear out
	 * any pending interrupts (from e.g. firmware based drives) and reset
	 * the interrupt handling registers.
	 */
	void (*irq_preinstall) (struct drm_device *dev);

	/**
	 * @irq_postinstall:
	 *
	 * Optional callback used by drm_irq_install() which is called after
	 * the interrupt handler is registered. This should be used to enable
	 * interrupt generation in the hardware.
	 */
	int (*irq_postinstall) (struct drm_device *dev);

	/**
	 * @irq_uninstall:
	 *
	 * Optional callback used by drm_irq_uninstall() which is called before
	 * the interrupt handler is unregistered. This should be used to disable
	 * interrupt generation in the hardware.
	 */
	void (*irq_uninstall) (struct drm_device *dev);

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
	 * @gem_free_object_unlocked: deconstructor for drm_gem_objects
	 *
	 * This is deprecated and should not be used by new drivers. Use
	 * &drm_gem_object_funcs.free instead.
	 */
	void (*gem_free_object_unlocked) (struct drm_gem_object *obj);

	/**
	 * @gem_open_object:
	 *
	 * This callback is deprecated in favour of &drm_gem_object_funcs.open.
	 *
	 * Driver hook called upon gem handle creation
	 */
	int (*gem_open_object) (struct drm_gem_object *, struct drm_file *);

	/**
	 * @gem_close_object:
	 *
	 * This callback is deprecated in favour of &drm_gem_object_funcs.close.
	 *
	 * Driver hook called upon gem handle release
	 */
	void (*gem_close_object) (struct drm_gem_object *, struct drm_file *);

	/**
	 * @gem_create_object: constructor for gem objects
	 *
	 * Hook for allocating the GEM object struct, for use by the CMA and
	 * SHMEM GEM helpers.
	 */
	struct drm_gem_object *(*gem_create_object)(struct drm_device *dev,
						    size_t size);
	/**
	 * @prime_handle_to_fd:
	 *
	 * Main PRIME export function. Should be implemented with
	 * drm_gem_prime_handle_to_fd() for GEM based drivers.
	 *
	 * For an in-depth discussion see :ref:`PRIME buffer sharing
	 * documentation <prime_buffer_sharing>`.
	 */
	int (*prime_handle_to_fd)(struct drm_device *dev, struct drm_file *file_priv,
				uint32_t handle, uint32_t flags, int *prime_fd);
	/**
	 * @prime_fd_to_handle:
	 *
	 * Main PRIME import function. Should be implemented with
	 * drm_gem_prime_fd_to_handle() for GEM based drivers.
	 *
	 * For an in-depth discussion see :ref:`PRIME buffer sharing
	 * documentation <prime_buffer_sharing>`.
	 */
	int (*prime_fd_to_handle)(struct drm_device *dev, struct drm_file *file_priv,
				int prime_fd, uint32_t *handle);
	/**
	 * @gem_prime_export:
	 *
	 * Export hook for GEM drivers. Deprecated in favour of
	 * &drm_gem_object_funcs.export.
	 */
	struct dma_buf * (*gem_prime_export)(struct drm_gem_object *obj,
					     int flags);
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
	 * @gem_prime_pin:
	 *
	 * Deprecated hook in favour of &drm_gem_object_funcs.pin.
	 */
	int (*gem_prime_pin)(struct drm_gem_object *obj);

	/**
	 * @gem_prime_unpin:
	 *
	 * Deprecated hook in favour of &drm_gem_object_funcs.unpin.
	 */
	void (*gem_prime_unpin)(struct drm_gem_object *obj);


	/**
	 * @gem_prime_get_sg_table:
	 *
	 * Deprecated hook in favour of &drm_gem_object_funcs.get_sg_table.
	 */
	struct sg_table *(*gem_prime_get_sg_table)(struct drm_gem_object *obj);

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
	 * @gem_prime_vmap:
	 *
	 * Deprecated vmap hook for GEM drivers. Please use
	 * &drm_gem_object_funcs.vmap instead.
	 */
	void *(*gem_prime_vmap)(struct drm_gem_object *obj);

	/**
	 * @gem_prime_vunmap:
	 *
	 * Deprecated vunmap hook for GEM drivers. Please use
	 * &drm_gem_object_funcs.vunmap instead.
	 */
	void (*gem_prime_vunmap)(struct drm_gem_object *obj, void *vaddr);

	/**
	 * @gem_prime_mmap:
	 *
	 * mmap hook for GEM drivers, used to implement dma-buf mmap in the
	 * PRIME helpers.
	 *
	 * FIXME: There's way too much duplication going on here, and also moved
	 * to &drm_gem_object_funcs.
	 */
	int (*gem_prime_mmap)(struct drm_gem_object *obj,
				struct vm_area_struct *vma);

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
	// 用于创建gem对象，并分配物理buffer
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
	 * @dumb_destroy:
	 *
	 * This destroys the userspace handle for the given dumb backing storage buffer.
	 * Since buffer objects must be reference counted in the kernel a buffer object
	 * won't be immediately freed if a framebuffer modeset object still uses it.
	 *
	 * Called by the user via ioctl.
	 *
	 * The default implementation is drm_gem_dumb_destroy(). GEM based drivers
	 * must not overwrite this.
	 *
	 * Returns:
	 *
	 * Zero on success, negative errno on failure.
	 */
	int (*dumb_destroy)(struct drm_file *file_priv,
			    struct drm_device *dev,
			    uint32_t handle);

	/**
	 * @gem_vm_ops: Driver private ops for this object
	 *
	 * For GEM drivers this is deprecated in favour of
	 * &drm_gem_object_funcs.vm_ops.
	 */
	const struct vm_operations_struct *gem_vm_ops;

	/** @major: driver major number */
	int major;
	/** @minor: driver minor number */
	int minor;
	/** @patchlevel: driver patch level */
	int patchlevel;
	/** @name: driver name */
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
	// 描述驱动特性
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

	/* Everything below here is for legacy driver, never use! */
	/* private: */

	/* List of devices hanging off this driver with stealth attach. */
	struct list_head legacy_dev_list;
	int (*firstopen) (struct drm_device *);
	void (*preclose) (struct drm_device *, struct drm_file *file_priv);
	int (*dma_ioctl) (struct drm_device *dev, void *data, struct drm_file *file_priv);
	int (*dma_quiescent) (struct drm_device *);
	int (*context_dtor) (struct drm_device *dev, int context);
	u32 (*get_vblank_counter)(struct drm_device *dev, unsigned int pipe);
	int (*enable_vblank)(struct drm_device *dev, unsigned int pipe);
	void (*disable_vblank)(struct drm_device *dev, unsigned int pipe);
	int dev_priv_size;
};
```

### phytium drm dc驱动
phytium E2000 DC控制器设备树描述
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

struct phytium_display_private结构体定义
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

struct phytium_device_info结构体定义
```c
struct phytium_device_info {
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

struct phytium_dp_device结构体定义
```c
struct phytium_dp_device {
	struct drm_device *dev;
	struct drm_encoder encoder;
	struct drm_connector connector;
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

	int *source_rates;
	int num_source_rates;
	int sink_rates[DP_MAX_SUPPORTED_RATES];
	int num_sink_rates;
	int common_rates[DP_MAX_SUPPORTED_RATES];
	int num_common_rates;

	int source_max_lane_count;
	int sink_max_lane_count;
	int common_max_lane_count;

	int max_link_rate;
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

phytium crtc相关结构体定义
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

phytium_plane结构体定义
```c
// drivers/gpu/drm/phytium/phytium_plane.h
struct phytium_plane {
	struct drm_plane base;
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

	ret = phytium_dp_hw_init(phytium_dp);
	if (ret) {
		DRM_ERROR("failed to initialize dp %d\n", phytium_dp->port);
		goto failed_init_dp;
	}

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

3. 打开drm驱动调试信息，增加内核启动参数`drm.debug=0x1f`

```bash
plane[31]: primary 0
        crtc=phys_pipe 0
        fb=60
                allocated by = gnome-shell
                refcount=3
                format=XR24 little-endian (0x34325258)
                modifier=0x0
                size=1920x1080
                layers:
                        size[0]=1920x1080
                        pitch[0]=7680
                        offset[0]=0
                        obj[0]:
                                name=0
                                refcount=1
                                start=00000000
                                size=8355840
                                imported=yes
        crtc-pos=1920x1080+0+0
        src-pos=1920.000000x1080.000000+0.000000+0.000000
        rotation=1
        normalized-zpos=0
        color-encoding=ITU-R BT.601 YCbCr
        color-range=YCbCr limited range
plane[34]: cursor 0
        crtc=phys_pipe 0
        fb=63
                allocated by = gnome-shell
                refcount=2
                format=AR24 little-endian (0x34325241)
                modifier=0x0
                size=32x32
                layers:
                        size[0]=32x32
                        pitch[0]=128
                        offset[0]=0
                        obj[0]:
                                name=0
                                refcount=3
                                start=00100fd2
                                size=4096
                                imported=no
        crtc-pos=32x32+868+165
        src-pos=32.000000x32.000000+0.000000+0.000000
        rotation=1
        normalized-zpos=0
        color-encoding=ITU-R BT.601 YCbCr
        color-range=YCbCr limited range
```


plane[31]: primary 0
        crtc=phys_pipe 0
        fb=54
                allocated by = gnome-shell
                refcount=3
                format=XR24 little-endian (0x34325258)
                modifier=0x0
                size=1280x720
                layers:
                        size[0]=1280x720
                        pitch[0]=5120
                        offset[0]=0
                        obj[0]:
                                name=0
                                refcount=2
                                start=001013b1
                                size=3932160
                                imported=no
        crtc-pos=1280x720+0+0
        src-pos=1280.000000x720.000000+0.000000+0.000000
        rotation=1
        normalized-zpos=0
        color-encoding=ITU-R BT.601 YCbCr
        color-range=YCbCr limited range
plane[34]: cursor 0
        crtc=phys_pipe 0
        fb=52
                allocated by = gnome-shell
                refcount=2
                format=AR24 little-endian (0x34325241)
                modifier=0x0
                size=32x32
                layers:
                        size[0]=32x32
                        pitch[0]=128
                        offset[0]=0
                        obj[0]:
                                name=0
                                refcount=3
                                start=00100ff0
                                size=4096
                                imported=no
        crtc-pos=32x32+714+172
        src-pos=32.000000x32.000000+0.000000+0.000000
        rotation=1
        normalized-zpos=0
        color-encoding=ITU-R BT.601 YCbCr
        color-range=YCbCr limited range

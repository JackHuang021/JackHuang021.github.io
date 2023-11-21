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
KMS主要负责两个功能：显示参数、显示控制，这两个基本功能是显示驱动必须具备的能力，在DRM框架下，为了将这两部分适配符合现代显示设备逻辑，又分出了几部分子模块配合框架

##### Plane
基本的显示控制单位，每个图像拥有一个Plane，Plane的属性控制着图像的显示区域、图像翻转、色彩混合方式等，最终图像经过Plane并通过CRTC组件，得到多个图像的混合显示或单独显示的功能

##### CRTC
CRTC的工作，就是负责把要显示的图像，转化为底层硬件层面上的具体时序要求，还负责着帧切换、电源控制、色彩调整等

##### Encoder
Encoder的工作是负责电源管理、视频输出格式封装，比如要将视频输出到HDMI、MIPI等

##### Connector
Connector连接器负责硬件设备的接入、屏参数获取等

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
```

### struct drm_device
```c
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

#if IS_ENABLED(CONFIG_DRM_LEGACY)
    struct list_head

#endif
};
```

### phytium drm dc驱动

drm_driver结构体
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
static int phytium_platform_probe(struct platform_device *pdev)
{
	struct phytium_display_private *priv = NULL;
	struct drm_device *dev = NULL;
	int ret = 0;

    // 注册一个drm_device
	dev = drm_dev_alloc(&phytium_display_drm_driver, &pdev->dev);
	if (IS_ERR(dev)) {
		DRM_ERROR("failed to allocate drm_device\n");
		return PTR_ERR(dev);
	}

	dev_set_drvdata(&pdev->dev, dev);
	dma_set_mask(&pdev->dev, DMA_BIT_MASK(40));

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


### DRM调试
针对xorg的显示问题排查步骤

1. 查看xorg log，`cat /var/log/Xorg.0.log`，从log信息中查找(WW)和(EE)相关log

2. 查看drm sysfs目录信息`/sys/class/drm`，

3. 打开drm驱动调试信息，增加内核启动参数`drm.debug=0x1f`
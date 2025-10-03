.. _accessing_land_parameters:

============================================
在aero_model中获取陆面参数 (Accessing Land Surface Parameters in Aerosol Models)
============================================

概述 (Overview)
================

本文档介绍如何在CESM的气溶胶模式(aero_model)中获取陆面参数，例如叶面积指数(LAI, Leaf Area Index)等陆面模型(CLM)输出的参数。

This document explains how to access land surface parameters, such as Leaf Area Index (LAI), 
from the land model (CLM) in CESM's aerosol models.

CESM组件耦合机制 (CESM Component Coupling Mechanism)
====================================================

CESM使用耦合器(coupler/driver)在各个组件之间交换数据。气溶胶模式通常作为大气模式(CAM)的一部分运行，
可以通过耦合器从陆面模式(CLM)获取参数。

CESM uses a coupler/driver to exchange data between components. Aerosol models typically run 
as part of the atmosphere model (CAM) and can receive parameters from the land model (CLM) 
through the coupler.

获取LAI的方法 (Methods to Access LAI)
=====================================

方法1: 通过耦合器字段 (Method 1: Through Coupler Fields)
--------------------------------------------------------

陆面模式(CLM)会将LAI等参数通过耦合器传递给大气模式。在气溶胶模式中，可以通过以下步骤获取：

The land model (CLM) passes parameters like LAI to the atmosphere model through the coupler. 
In your aerosol model, you can access these through the following steps:

1. **声明耦合器字段 (Declare Coupler Field)**

   在Fortran代码中声明需要接收的字段：

   .. code-block:: fortran

      ! 在模块声明部分
      use physics_buffer, only: physics_buffer_desc, pbuf_get_field
      
      ! 声明LAI相关变量
      real(r8), pointer :: lai(:,:)  ! leaf area index
      integer :: lai_idx              ! physics buffer index for LAI

2. **初始化阶段注册字段 (Register Field During Initialization)**

   在初始化子程序中注册LAI字段：

   .. code-block:: fortran

      subroutine aero_model_init()
         use physics_buffer, only: pbuf_add_field, dtype_r8
         
         ! 在physics buffer中添加LAI字段
         call pbuf_add_field('LAI', 'physpkg', dtype_r8, (/pcols,pver/), lai_idx)
         
      end subroutine aero_model_init

3. **运行时获取LAI数据 (Get LAI Data at Runtime)**

   在时间步循环中获取LAI数据：

   .. code-block:: fortran

      subroutine aero_model_run(state, pbuf, dt)
         use physics_types, only: physics_state
         use physics_buffer, only: physics_buffer_desc, pbuf_get_field
         
         type(physics_state), intent(in) :: state
         type(physics_buffer_desc), pointer :: pbuf(:)
         real(r8), intent(in) :: dt
         
         ! 局部变量
         real(r8), pointer :: lai(:,:)
         integer :: ncol, i, k
         
         ncol = state%ncol
         
         ! 从physics buffer获取LAI
         call pbuf_get_field(pbuf, lai_idx, lai)
         
         ! 使用LAI数据进行气溶胶计算
         do i = 1, ncol
            do k = 1, pver
               ! 在此处使用lai(i,k)进行计算
               ! Your aerosol calculations using lai(i,k)
            end do
         end do
         
      end subroutine aero_model_run


方法2: 通过表面数据接口 (Method 2: Through Surface Data Interface)
------------------------------------------------------------------

如果需要直接从CLM的表面数据获取LAI，可以使用CLM的历史场接口：

If you need to directly access LAI from CLM's surface data:

.. code-block:: fortran

   subroutine get_lai_from_clm()
      use shr_kind_mod, only: r8 => shr_kind_r8
      use cam_history, only: outfld
      use pio, only: file_desc_t, iosystem_desc_t
      
      ! 从CLM历史输出中读取LAI
      ! 注意：这需要在namelist中配置CLM输出LAI
      
   end subroutine get_lai_from_clm


配置要求 (Configuration Requirements)
=====================================

确保LAI传递到大气模式 (Ensure LAI is Passed to Atmosphere)
----------------------------------------------------------

1. **配置CLM输出 (Configure CLM Output)**

   在 ``user_nl_clm`` 文件中添加：

   .. code-block:: text

      hist_fincl1 = 'TLAI'

   其中 ``TLAI`` 是总叶面积指数 (Total Leaf Area Index)。

2. **配置耦合器字段 (Configure Coupler Fields)**

   确保在耦合器配置中启用了陆面到大气的LAI传递。这通常在 
   ``$CASEROOT/CaseDocs/drv_flds_in`` 文件中配置。

3. **检查可用字段 (Check Available Fields)**

   运行模式后，检查耦合器日志文件以确认LAI字段已成功传递：

   .. code-block:: bash

      grep -i "lai" $CASEROOT/run/cesm.log.*


常用陆面参数 (Common Land Surface Parameters)
=============================================

除了LAI，以下是其他常用的陆面参数：

Besides LAI, here are other commonly used land surface parameters:

.. csv-table:: 常用陆面参数 (Common Land Parameters)
   :header: "参数名称 (Parameter)", "CLM变量 (CLM Variable)", "描述 (Description)"
   :widths: 20, 20, 60

   "叶面积指数 (LAI)", "TLAI", "总叶面积指数 Total Leaf Area Index"
   "茎面积指数 (SAI)", "TSAI", "总茎面积指数 Total Stem Area Index"
   "植被覆盖度 (Vegetation Fraction)", "FPSN", "光合作用叶片分数 Sunlit fraction of canopy"
   "地表反照率 (Surface Albedo)", "ALB", "地表反照率 Surface albedo"
   "土壤湿度 (Soil Moisture)", "H2OSOI", "土壤液态水含量 Soil liquid water"
   "积雪深度 (Snow Depth)", "SNOWDP", "积雪深度 Snow depth"


代码示例 (Code Example)
=======================

完整的气溶胶模块示例 (Complete Aerosol Module Example)
------------------------------------------------------

完整的Fortran示例代码可以在 ``doc/examples/aero_model_lai_interface.F90`` 中找到。

A complete Fortran example can be found in ``doc/examples/aero_model_lai_interface.F90``.

以下是简化版本的示例：

Here is a simplified example:

.. code-block:: fortran

   module aero_model_lai
   !-----------------------------------------------------------------------
   ! 气溶胶模式LAI接口示例
   ! Example aerosol model interface for LAI access
   !-----------------------------------------------------------------------
   
   use shr_kind_mod,    only: r8 => shr_kind_r8
   use ppgrid,          only: pcols, pver, begchunk, endchunk
   use physics_types,   only: physics_state
   use physics_buffer,  only: physics_buffer_desc, pbuf_get_field, pbuf_add_field, dtype_r8
   
   implicit none
   private
   save
   
   public :: aero_model_lai_init
   public :: aero_model_lai_run
   
   ! 模块变量
   integer :: lai_idx = 0    ! LAI physics buffer index
   
   contains
   
   !-----------------------------------------------------------------------
   subroutine aero_model_lai_init()
   !-----------------------------------------------------------------------
   ! 初始化LAI接口
   ! Initialize LAI interface
   !-----------------------------------------------------------------------
   
      ! 注册LAI字段到physics buffer
      call pbuf_add_field('LAI', 'physpkg', dtype_r8, (/pcols,pver/), lai_idx)
      
      write(6,*) 'aero_model_lai_init: LAI interface initialized'
      
   end subroutine aero_model_lai_init
   
   !-----------------------------------------------------------------------
   subroutine aero_model_lai_run(state, pbuf)
   !-----------------------------------------------------------------------
   ! 获取并使用LAI数据
   ! Get and use LAI data
   !-----------------------------------------------------------------------
   
      type(physics_state), intent(in) :: state
      type(physics_buffer_desc), pointer :: pbuf(:)
      
      ! 局部变量
      real(r8), pointer :: lai(:,:)
      real(r8) :: lai_column_avg
      integer :: ncol, i, k
      
      ncol = state%ncol
      
      ! 从physics buffer获取LAI
      call pbuf_get_field(pbuf, lai_idx, lai)
      
      ! 处理LAI数据
      do i = 1, ncol
         lai_column_avg = 0.0_r8
         do k = 1, pver
            lai_column_avg = lai_column_avg + lai(i,k)
         end do
         lai_column_avg = lai_column_avg / real(pver, r8)
         
         ! 在此使用LAI进行气溶胶计算
         ! Use LAI for aerosol calculations here
         ! 例如：调整干沉降速率、计算生物气溶胶排放等
         ! For example: adjust dry deposition velocity, calculate bioaerosol emissions, etc.
         
      end do
      
   end subroutine aero_model_lai_run
   
   end module aero_model_lai


调试和验证 (Debugging and Verification)
========================================

验证LAI数据 (Verify LAI Data)
------------------------------

1. **添加诊断输出 (Add Diagnostic Output)**

   .. code-block:: fortran

      ! 输出LAI统计信息
      if (masterproc) then
         write(6,*) 'LAI min/max/mean:', minval(lai), maxval(lai), sum(lai)/size(lai)
      end if

2. **输出到历史文件 (Output to History Files)**

   .. code-block:: fortran

      use cam_history, only: outfld
      
      ! 将LAI输出到CAM历史文件
      call outfld('AERO_LAI', lai, pcols, lchnk)

3. **检查耦合器日志 (Check Coupler Logs)**

   查看运行目录中的耦合器日志文件，确认字段传递正常。


参考资料 (References)
=====================

1. CESM耦合器文档: http://esmci.github.io/cime/versions/master/html/driver_cpl/index.html
2. CLM用户指南: http://www.cesm.ucar.edu/models/cesm2/land/
3. CAM接口文档: http://www.cesm.ucar.edu/models/cesm2/atmosphere/
4. Physics Buffer接口: 查看 ``$CAMROOT/src/physics/cam/physics_buffer.F90``


常见问题 (FAQ)
===============

Q: LAI字段为零或无效值？
------------------------

A: 检查以下几点：

1. 确认CLM配置正确输出LAI
2. 检查耦合器字段映射是否正确
3. 验证physics buffer初始化顺序
4. 确认运行的compset包含活动的陆面模式(CLM)

Q: 如何获取特定植被类型的LAI？
-------------------------------

A: CLM提供了分植被类型(PFT-specific)的LAI。需要：

1. 在CLM namelist中配置输出PFT级别的LAI
2. 在气溶胶模式中添加相应的接收数组
3. 通过耦合器传递多维LAI数组

Q: 性能影响？
-------------

A: 通过耦合器传递字段的性能影响通常很小。如果需要高频率访问，
可以考虑在本地缓存LAI数据。


技术支持 (Technical Support)
=============================

如有问题，请访问CESM论坛：https://bb.cgd.ucar.edu/cesm

For questions, please visit the CESM forum: https://bb.cgd.ucar.edu/cesm


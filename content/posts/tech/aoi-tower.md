---
title: "四叉树Lod灯塔AOI"
date: 2022-10-28T01:30:29+08:00
lastmod: 2022-10-28T01:30:29+08:00
author: ["frog"]
keywords:
-
categories:
-
tags:
-  白皮书
description: "用c代码写的四叉树lod灯塔AOI视野同步，此AOI已经用于frog-game-frame框架"
weight:
draft: false # 是否为草稿
comments: true
reward: true # 打赏
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: "posts/tech/aoi-tower/image-20221028105101531.png" #图片路径例如：posts/tech/123/123.png
    caption: "" #图片底部描述
    alt: ""
    relative: false
---


## 一些字段的解释

1. 观察者：我可以观察到那些人。
2. 被观察者：那些人能观察到自己。
3. \#define WATCHER_MODE 0x01 观察者模式
4. \#define MARKER_MODE 0x02 被观察者模式

## 灯塔相关[结构体]

**1：灯塔区域结构**

```c
struct towerSpace_s
{
	void (*callback)(void*pUserData,bool bAddTo,uint64_t watcher, uint64_t marker); -- 回调函数
	void*			pUserData; //用户信息
	float			fMin[2]; //最小位置
	float			fGridLength[3];//网格x y,z方向长度
	float			fMovefRange;//移动范围
	int32_t			iSplitThreshold;//拆分阈值
	int32_t 		iMaxWidth; //最大宽度
	int32_t 		iMaxHeight;//最大高度
    int32_t*  		pGrids;//网格数据
	tower_tt*		pTowers;//灯塔数据
	int32_t			iTowerNext;//下一个灯塔id
	int32_t			iTowerCapacity;//灯塔容量
	aoiObj_tt* 		pSlotObj;//格子里面对象
	int32_t			iSlotIndex;//格子索引
	int32_t     	iSlotCapacity;//格子容量
};
```

2：**灯塔信息结构**

```c
typedef struct tower_s
{
	aoi_tree_tt watcher; //灯塔观察者[用来存储观察到的对象]
	aoi_tree_tt	marker;//灯塔被观察者
	int32_t     iMarkerCount;//被观察者数量
	int32_t  	iFirstChildId;//第一个儿子节点索引
} tower_tt;
```

**3：灯塔划分后的节点结构【四个儿子节点】**

```c
typedef struct aoiNode_s
{
	RB_ENTRY(aoiNode_s) entry; //实体
	int32_t				iId; //id
} aoiNode_tt;
```

**4：灯塔里面对象结构**

```c
typedef struct aoiObj_s
{ 
	int32_t		iId; // id
	int32_t 	iMode; // 模式(MARKER_MODE:被观察者模式 WATCHER_MODE:观察模式)
	uint64_t    uiMask;//掩码
	uint64_t    uiUserData; //用户数据
	float		fViewRadius; // 视野半径
	float 		last[3]; //上一个xyz位置
	float 		pos[3];//当前xyz位置
} aoiObj_tt;
```

## 灯塔AOI相关的一些操作函数

```c
int32_t luaopen_laoi(lua_State *L)
{
#ifdef luaL_checkversion
	luaL_checkversion(L);
#endif
	registerTowerSpaceL(L);

	luaL_Reg lualib_funcs[] =
	{
		{"createAoiSpace",		lcreateAoiSpace},//创建aoi区域
		{NULL, NULL}
	};
	luaL_newlib(L, lualib_funcs);
	return 1;
}

int32_t registerTowerSpaceL(struct lua_State *L)
{
	luaL_newmetatable(L, "towerSpace");
	lua_pushvalue(L, -1);
	lua_setfield(L, -2, "__index");

	struct luaL_Reg lua_towerSpaceFuncs[] = 
	{
		{"setCallback",		laoi_setCallback}, //设置回调函数
		{"addObj",			laoi_addObj},//增加一个实体对象
		{"removeObj",		laoi_removeObj},//移除一个实体对象
		{"updateObjMask",	laoi_updateObjMask},//更新对象的mask[0x01:观察者 0x02:被观察者]
		{"updateObjPos",	laoi_updateObjPos},//更新对象的pos
		{"addObjWatcher",	laoi_addObjWatcher},//增加对象到相应的观察容器
		{"removeObjWatcher",laoi_removeObjWatcher},//从相应的观察容器移除对象
		{"addObjMarker",	laoi_addObjMarker},//增加对象到被观察者
		{"removeObjMarker",	laoi_removeObjMarker},//移除对象到被观察者
		{"__gc", 			laoi_towerSpace_gc},//此区域进行GC回收
		{NULL, NULL}
	};

	luaL_setfuncs(L, lua_towerSpaceFuncs, 0);
	return 1;
}
```

## [四叉树]lod示意图

![image-20221028105101531](image-20221028105101531.png)

1. 黑色大框是AOI的区域大小
2. 每个正方形块上面都有一个灯塔
3. 暂时定的最多分裂3层
4. 黑色的原点是场景内的实体

## 灯塔AOI一些关键函数[具体代码在frog-game-server框架]

**1. 更新观察者集合**

```c
inline static void changeAoiObjWatcher(towerSpace_tt* pTowerSpace,aoiObj_tt* pObj)
{
	float bmin[3];
	float bmax[3];

	bmin[0] = pObj->last[0] - pObj->fViewRadius;
	bmin[2] = pObj->last[2] - pObj->fViewRadius;
	bmax[0] = pObj->last[0] + pObj->fViewRadius;
	bmax[2] = pObj->last[2] + pObj->fViewRadius;
	
	int32_t minxLast = 0;
	int32_t minyLast = 0;
	int32_t maxxLast = 0;
	int32_t maxyLast = 0;

	calcGridLodLoc(pTowerSpace, 2,bmin, &minxLast, &minyLast);
	calcGridLodLoc(pTowerSpace, 2,bmax, &maxxLast, &maxyLast);

	minxLast = minxLast > 0 ? minxLast : 0;
	minyLast = minyLast > 0 ? minyLast : 0;
	maxxLast = maxxLast < pTowerSpace->iMaxWidth * 4 ? maxxLast : pTowerSpace->iMaxWidth*4 - 1;
	maxyLast = maxyLast < pTowerSpace->iMaxHeight * 4 ? maxyLast : pTowerSpace->iMaxHeight*4 - 1;


	bmin[0] = pObj->pos[0] - pObj->fViewRadius;
	bmin[2] = pObj->pos[2] - pObj->fViewRadius;

	bmax[0] = pObj->pos[0] + pObj->fViewRadius;
	bmax[2] = pObj->pos[2] + pObj->fViewRadius;

	int32_t minx = 0;
	int32_t miny = 0;
	int32_t maxx = 0;
	int32_t maxy = 0;

	calcGridLodLoc(pTowerSpace, 2,bmin, &minx, &miny);
	calcGridLodLoc(pTowerSpace, 2,bmax, &maxx, &maxy);

	minx = minx > 0 ? minx : 0;
	miny = miny > 0 ? miny : 0;
	maxx = maxx < pTowerSpace->iMaxWidth*4 ? maxx : pTowerSpace->iMaxWidth*4 - 1;
	maxy = maxy < pTowerSpace->iMaxHeight*4 ? maxy : pTowerSpace->iMaxHeight*4 - 1;
	
	//是否重合
	if(isOverlap(minx,miny,maxx,maxy,minxLast,minyLast,maxxLast,maxyLast))
	{
		int32_t iMinX = minx < minxLast ? minx : minxLast;
		int32_t iMinY = miny < minyLast ? miny : minyLast;
		int32_t iMaxX = maxx >= maxxLast ? maxx : maxxLast;
		int32_t iMaxY = maxy >= maxyLast ? maxy : maxyLast;

		int32_t iChanged = 0;

		//往上找到最大的网格块
		//为什么是iMinY/4 是因为除以4就像四叉树一样找最上面的父节点的索引值一样
		for (int32_t iY = iMinY/4; iY < (iMaxY+3)/4; ++iY) 
		{
			for (int32_t iX = iMinX/4; iX < (iMaxX+3)/4; ++iX)
			{
				iChanged = 0;
				if(isInInside(iX*4,iY*4,iX*4+3,iY*4+3,minxLast,minyLast,maxxLast,maxyLast))
				{
					iChanged = 0x1;
				}

				if(isInInside(iX*4,iY*4,iX*4+3,iY*4+3,minx,miny,maxx,maxy))
				{
					iChanged |= 0x2;
				}

				switch (iChanged)
				{
				case 0x1:
					{
						removeGridWatcher(pTowerSpace,pObj,iX,iY);
					}
					break;
				case 0x2:
					{
						insertGridWatcher(pTowerSpace,pObj,iX,iY,pObj->pos);
					}
					break;
				case 0x3:
					{
						int32_t iTowerId = pTowerSpace->pGrids[iX + iY * pTowerSpace->iMaxWidth];
						assert(iTowerId != -1);
						tower_tt* pTower = pTowerSpace->pTowers + iTowerId;
						if (pTower->iFirstChildId == -1)
						{
							continue;
						}

						for (int32_t ly = 0; ly < 2; ly++)
						{
							for (int32_t lx = 0; lx < 2; lx++)
							{
								iChanged = 0;
								if (isInInside(iX * 4 + lx * 2, iY * 4 + ly * 2, iX * 4 + lx * 2 + 1, iY * 4 + ly * 2 + 1, minxLast, minyLast, maxxLast, maxyLast))
								{
									iChanged = 0x1;
								}

								if (isInInside(iX * 4 + lx * 2, iY * 4 + ly * 2, iX * 4 + lx * 2 + 1, iY * 4 + ly * 2 + 1, minx, miny, maxx, maxy))
								{
									iChanged |= 0x2;
								}

								switch (iChanged)
								{
								case 0x1:
									{
										removeLodWatcher(pTowerSpace, iTowerId, pObj, iX, iY, iX * 4 + lx * 2, iY * 4 + ly * 2);
									}
									break;
								case 0x2:
									{
										insertLodWatcher(pTowerSpace, iTowerId, pObj, iX, iY, iX * 4 + lx * 2, iY * 4 + ly * 2);
									}
									break;
								case 0x3:
									{
										tower_tt* pLodTower = pTowerSpace->pTowers + pTower->iFirstChildId + ly * 2 + lx;
										if (pLodTower->iFirstChildId == -1)
										{
											continue;
										}

										for (int32_t l2y = 0; l2y < 2; l2y++)
										{
											for (int32_t l2x = 0; l2x < 2; l2x++)
											{
												iChanged = 0;
												if (isInRect(iX * 4 + lx * 2+l2x, iY * 4 + ly * 2+l2y,  minxLast, minyLast, maxxLast, maxyLast))
												{
													iChanged = 0x1;
												}

												if (isInRect(iX * 4 + lx * 2+l2x, iY * 4 + ly * 2+l2y, minx, miny, maxx, maxy))
												{
													iChanged |= 0x2;
												}

												switch (iChanged)
												{
												case 0x1:
													{
														tower_tt* pLod2Tower = pTowerSpace->pTowers + pLodTower->iFirstChildId + l2y * 2 + l2x;
														aoiNode_tt findNode;
														findNode.iId = pObj->iId;
														aoiNode_tt* pT = RB_FIND(aoi_tree_s, &pLod2Tower->watcher, &findNode);
														assert(pT);
														RB_REMOVE(aoi_tree_s, &pLod2Tower->watcher, pT);
														mem_free(pT);

														aoiNode_tt* pI;
														RB_FOREACH(pI, aoi_tree_s, &pLod2Tower->marker)
														{
															aoiObj_tt* pMarkerObj = pTowerSpace->pSlotObj + pI->iId;
															if ((pI->iId != pObj->iId) && (pObj->uiMask & pMarkerObj->uiMask))
															{
																pTowerSpace->callback(pTowerSpace->pUserData, false, pObj->uiUserData, pMarkerObj->uiUserData);
															}
														}
													}
													break;
												case 0x2:
													{
														tower_tt* pLod2Tower = pTowerSpace->pTowers + pLodTower->iFirstChildId + l2y * 2 + l2x;
														aoiNode_tt* pNode = mem_malloc(sizeof(aoiNode_tt));
														pNode->iId = pObj->iId;
														RB_INSERT(aoi_tree_s, &pLod2Tower->watcher, pNode);

														aoiNode_tt* pI;
														RB_FOREACH(pI, aoi_tree_s, &pLod2Tower->marker)
														{
															aoiObj_tt* pMarkerObj = pTowerSpace->pSlotObj + pI->iId;
															if ((pI->iId != pObj->iId) && (pObj->uiMask & pMarkerObj->uiMask))
															{
																pTowerSpace->callback(pTowerSpace->pUserData, true, pObj->uiUserData, pMarkerObj->uiUserData);
															}
														}
													}
													break;
												}
											}
										}
									}
									break;
								}
							}
						}
					}
					break;
				}
			}
		}
	}
	else
	{
		for (int32_t iY = minyLast / 4; iY <= (maxyLast + 3) / 4; ++iY)
		{
			for (int32_t iX = minxLast / 4; iX <= (maxxLast + 3) / 4; ++iX)
			{
				removeGridWatcher(pTowerSpace, pObj, iX, iY);
			}
		}

		for (int32_t iY = miny/4; iY <= (maxy+3)/4; ++iY)
		{
			for (int32_t iX = minx/4; iX <= (maxx+3)/4; ++iX)
			{
				insertGridWatcher(pTowerSpace,pObj,iX,iY,pObj->pos);
			}
		}
	}	
}
```

**2. 把被观察者转入观察者容器**

```c
inline static void changeAoiObjMaskToWatcher(towerSpace_tt* pTowerSpace,aoiObj_tt* pObj,int32_t iX,int32_t iY,uint64_t uiMask)
{
	int32_t iTowerId = pTowerSpace->pGrids[iX+iY*pTowerSpace->iMaxWidth];
    assert(iTowerId != -1);
    
	tower_tt* pTower = pTowerSpace->pTowers + iTowerId;
    if (pTower->iFirstChildId == -1)
    {
		int32_t iChanged;
		aoiNode_tt* pI; 
		RB_FOREACH(pI,aoi_tree_s,&pTower->marker)
		{
			if(pI->iId != pObj->iId)
			{
				iChanged = 0;
				aoiObj_tt* pMarkerObj = pTowerSpace->pSlotObj + pI->iId;
				if(pMarkerObj->iMode&pObj->uiMask)
				{
					iChanged = 0x1;
				}

				if(pMarkerObj->iMode&uiMask)
				{
					iChanged |= 0x2;
				}

				switch (iChanged)
				{
				case 0x1:
					{
						pTowerSpace->callback(pTowerSpace->pUserData,false,pObj->uiUserData,pMarkerObj->uiUserData);
					}
					break;
				case 0x2:
					{
						pTowerSpace->callback(pTowerSpace->pUserData,true,pObj->uiUserData,pMarkerObj->uiUserData);
					}
					break;
				}
			} 
		}
        return;
    }

    int32_t minGridX =iX*4;
    int32_t minGridY =iY*4;
    int32_t maxGridX =iX*4+3;
    int32_t maxGridY =iY*4+3;

    float bmin[3];
    float bmax[3];

    int32_t minx = 0;
    int32_t miny = 0;
    int32_t maxx = 0;
    int32_t maxy = 0;

    bmin[0] = pObj->last[0] - pObj->fViewRadius;
    bmin[2] = pObj->last[2] - pObj->fViewRadius;

	bmax[0] = pObj->last[0] + pObj->fViewRadius;
	bmax[2] = pObj->last[2] + pObj->fViewRadius;
    calcGridLodLoc(pTowerSpace, 2,bmin, &minx, &miny);
    calcGridLodLoc(pTowerSpace, 2,bmax, &maxx, &maxy);

    if(!isInInside(minGridX,minGridY,maxGridX,maxGridY,minx,miny,maxx,maxy))
    {
        for (int32_t y = 0; y < 2; y++)
        {
            for (int32_t x = 0; x < 2; x++)
            {
                tower_tt* pLodTower = pTowerSpace->pTowers + pTower->iFirstChildId + y*2+x;
                if(pLodTower->iFirstChildId == -1)
                {
					int32_t iChanged;
					aoiNode_tt* pI; 
					RB_FOREACH(pI,aoi_tree_s,&pLodTower->marker)
					{
						if(pI->iId != pObj->iId)
						{
							iChanged = 0;
							aoiObj_tt* pMarkerObj = pTowerSpace->pSlotObj + pI->iId;
							if(pMarkerObj->iMode&pObj->uiMask)
							{
								iChanged = 0x1;
							}

							if(pMarkerObj->iMode&uiMask)
							{
								iChanged |= 0x2;
							}

							switch (iChanged)
							{
							case 0x1:
								{
									pTowerSpace->callback(pTowerSpace->pUserData,false,pObj->uiUserData,pMarkerObj->uiUserData);
								}
								break;
							case 0x2:
								{
									pTowerSpace->callback(pTowerSpace->pUserData,true,pObj->uiUserData,pMarkerObj->uiUserData);
								}
								break;
							}
						}

					}
                }
                else
                {
                    if(!isInInside(minGridX+x*2,minGridY+y*2,minGridX+x*2+1,minGridY+y*2+1,minx,miny,maxx,maxy))
                    {
                        for (int32_t ly = 0; ly < 2; ly++)
                        {
                            for (int32_t lx = 0; lx < 2; lx++)
                            {
                                if(isInRect(minGridX+x*2+lx,minGridY+y*2+ly,minx,miny,maxx,maxy))
                                {
                                    tower_tt* pLod2Tower = pTowerSpace->pTowers + pLodTower->iFirstChildId + ly*2+lx;
									int32_t iChanged;
                                    aoiNode_tt* pI; 
                                    RB_FOREACH(pI,aoi_tree_s,&pLod2Tower->marker)
                                    {
										if(pI->iId != pObj->iId)
										{
											iChanged = 0;
											aoiObj_tt* pMarkerObj = pTowerSpace->pSlotObj + pI->iId;
											if(pMarkerObj->iMode&pObj->uiMask)
											{
												iChanged = 0x1;
											}

											if(pMarkerObj->iMode&uiMask)
											{
												iChanged |= 0x2;
											}

											switch (iChanged)
											{
											case 0x1:
												{
													pTowerSpace->callback(pTowerSpace->pUserData,false,pObj->uiUserData,pMarkerObj->uiUserData);
												}
												break;
											case 0x2:
												{
													pTowerSpace->callback(pTowerSpace->pUserData,true,pObj->uiUserData,pMarkerObj->uiUserData);
												}
												break;
											}
										}
										
                                    }
                                }
                            }
                        }
                    }
                    else
                    {
						int32_t iChanged;
                        aoiNode_tt* pI; 
                        for (int32_t i = 0; i < 4; i++)
                        {
                            tower_tt* pLod2Tower = pTowerSpace->pTowers + pLodTower->iFirstChildId+i;
                            RB_FOREACH(pI,aoi_tree_s,&pLod2Tower->marker)
                            {
								if(pI->iId != pObj->iId)
								{
									aoiObj_tt* pMarkerObj = pTowerSpace->pSlotObj + pI->iId;
									iChanged = 0;
									if(pMarkerObj->iMode&pObj->uiMask)
									{
										iChanged = 0x1;
									}

									if(pMarkerObj->iMode&uiMask)
									{
										iChanged |= 0x2;
									}

									switch (iChanged)
									{
									case 0x1:
										{
											pTowerSpace->callback(pTowerSpace->pUserData,false,pObj->uiUserData,pMarkerObj->uiUserData);
										}
										break;
									case 0x2:
										{
											pTowerSpace->callback(pTowerSpace->pUserData,true,pObj->uiUserData,pMarkerObj->uiUserData);
										}
										break;
									}
								}
                            }
                        }
                    }
                }
            }
        }
    }
    else
    {
		int32_t iChanged;
		aoiNode_tt* pI; 

        for (int32_t i = 0; i < 4; i++)
        {
            tower_tt* pLodTower = pTowerSpace->pTowers + pTower->iFirstChildId+i;
            if (pLodTower->iFirstChildId == -1)
            {
                RB_FOREACH(pI,aoi_tree_s,&pLodTower->marker)
                {
					if(pI->iId != pObj->iId)
					{
						aoiObj_tt* pMarkerObj = pTowerSpace->pSlotObj + pI->iId;
						iChanged = 0;
						if(pMarkerObj->iMode&pObj->uiMask)
						{
							iChanged = 0x1;
						}

						if(pMarkerObj->iMode&uiMask)
						{
							iChanged |= 0x2;
						}

						switch (iChanged)
						{
						case 0x1:
							{
								pTowerSpace->callback(pTowerSpace->pUserData,false,pObj->uiUserData,pMarkerObj->uiUserData);
							}
							break;
						case 0x2:
							{
								pTowerSpace->callback(pTowerSpace->pUserData,true,pObj->uiUserData,pMarkerObj->uiUserData);
							}
							break;
						}
					}
                }
            }
            else
            {
                for (int32_t j = 0; j < 4; j++)
                {
                    tower_tt* pLod2Tower = pTowerSpace->pTowers + pLodTower->iFirstChildId+j;
                    RB_FOREACH(pI,aoi_tree_s,&pLod2Tower->marker)
                    {
						if(pI->iId != pObj->iId)
						{
							aoiObj_tt* pMarkerObj = pTowerSpace->pSlotObj + pI->iId;
							iChanged = 0;
							if(pMarkerObj->iMode&pObj->uiMask)
							{
								iChanged = 0x1;
							}

							if(pMarkerObj->iMode&uiMask)
							{
								iChanged |= 0x2;
							}

							switch (iChanged)
							{
							case 0x1:
								{
									pTowerSpace->callback(pTowerSpace->pUserData,false,pObj->uiUserData,pMarkerObj->uiUserData);
								}
								break;
							case 0x2:
								{
									pTowerSpace->callback(pTowerSpace->pUserData,true,pObj->uiUserData,pMarkerObj->uiUserData);
								}
								break;
							}
						}
                    }
                }
            }
        }
    }
}
```

**3. 更新AOI的被观察者**

```c
void towerSpace_updateAoiObjMask(towerSpace_tt* pTowerSpace,int32_t iObjId,uint64_t uiMask)
{
	aoiObj_tt* pObj = pTowerSpace->pSlotObj + iObjId;
	assert(pObj->iId == iObjId);
	if(pObj->uiMask == uiMask)
	{
		return;
	}

	if(pObj->iMode&MARKER_MODE)
	{
		int32_t iX;
		int32_t iY;
		calcGridLoc(pTowerSpace,pObj->last,&iX,&iY);
		int32_t iTowerId = pTowerSpace->pGrids[iX+iY*pTowerSpace->iMaxWidth];
		assert(iTowerId != -1);
		tower_tt* pTower = pTowerSpace->pTowers + iTowerId;
		if(pTower->iFirstChildId == -1)
		{
			int32_t iChanged = 0;
			aoiNode_tt* pI; 
			RB_FOREACH(pI,aoi_tree_s,&pTower->watcher)
			{
				if(pI->iId != pObj->iId)
				{
					aoiObj_tt* pWatcherObj = pTowerSpace->pSlotObj + pI->iId;
					iChanged = 0;
					if(pWatcherObj->iMode&pObj->uiMask)
					{
						iChanged = 0x1;
					}

					if(pWatcherObj->iMode&uiMask)
					{
						iChanged |= 0x2;
					}

					switch (iChanged)
					{
					case 0x1:
						{
							pTowerSpace->callback(pTowerSpace->pUserData,false,pI->iId,pObj->iId);
						}
						break;
					case 0x2:
						{
							pTowerSpace->callback(pTowerSpace->pUserData,true,pI->iId,pObj->iId);
						}
						break;
					}
				}
			}
		}
		else
		{
			int32_t iLodX;
			int32_t iLodY;
			calcGridLodLoc(pTowerSpace,1,pObj->last,&iLodX,&iLodY);
			int32_t iTowerLodId = pTower->iFirstChildId + (iLodX -iX*2)+(iLodY - iY*2)*2;
			tower_tt* pLodTower = pTowerSpace->pTowers + iTowerLodId;
			if(pLodTower->iFirstChildId == -1)
			{
				int32_t iChanged = 0;
				aoiNode_tt* pI; 
				RB_FOREACH(pI,aoi_tree_s,&pTower->watcher)
				{
					if(pI->iId != pObj->iId)
					{
						aoiObj_tt* pWatcherObj = pTowerSpace->pSlotObj + pI->iId;
						iChanged = 0;
						if(pWatcherObj->iMode&pObj->uiMask)
						{
							iChanged = 0x1;
						}

						if(pWatcherObj->iMode&uiMask)
						{
							iChanged |= 0x2;
						}

						switch (iChanged)
						{
						case 0x1:
							{
								pTowerSpace->callback(pTowerSpace->pUserData,false,pI->iId,pObj->iId);
							}
							break;
						case 0x2:
							{
								pTowerSpace->callback(pTowerSpace->pUserData,true,pI->iId,pObj->iId);
							}
							break;
						}
					}
				}

				RB_FOREACH(pI,aoi_tree_s,&pLodTower->watcher)
				{
					if(pI->iId != pObj->iId)
					{
						aoiObj_tt* pWatcherObj = pTowerSpace->pSlotObj + pI->iId;
						iChanged = 0;
						if(pWatcherObj->iMode&pObj->uiMask)
						{
							iChanged = 0x1;
						}

						if(pWatcherObj->iMode&uiMask)
						{
							iChanged |= 0x2;
						}

						switch (iChanged)
						{
						case 0x1:
							{
								pTowerSpace->callback(pTowerSpace->pUserData,false,pI->iId,pObj->iId);
							}
							break;
						case 0x2:
							{
								pTowerSpace->callback(pTowerSpace->pUserData,true,pI->iId,pObj->iId);
							}
							break;
						}
					}
				}
			}
			else
			{
				int32_t iLod2X;
				int32_t iLod2Y;
				calcGridLodLoc(pTowerSpace,2,pObj->last,&iLod2X,&iLod2Y);
				int32_t iTowerLod2Id = pLodTower->iFirstChildId + (iLod2X -iLodX*2)+(iLod2Y - iLodY*2)*2;
				tower_tt* pLod2Tower = pTowerSpace->pTowers + iTowerLod2Id;

				int32_t iChanged = 0;
				aoiNode_tt* pI; 
				RB_FOREACH(pI,aoi_tree_s,&pTower->watcher)
				{
					if(pI->iId != pObj->iId)
					{
						aoiObj_tt* pWatcherObj = pTowerSpace->pSlotObj + pI->iId;
						iChanged = 0;
						if(pWatcherObj->iMode&pObj->uiMask)
						{
							iChanged = 0x1;
						}

						if(pWatcherObj->iMode&uiMask)
						{
							iChanged |= 0x2;
						}

						switch (iChanged)
						{
						case 0x1:
							{
								pTowerSpace->callback(pTowerSpace->pUserData,false,pI->iId,pObj->iId);
							}
							break;
						case 0x2:
							{
								pTowerSpace->callback(pTowerSpace->pUserData,true,pI->iId,pObj->iId);
							}
							break;
						}
					}
					
				}

				RB_FOREACH(pI,aoi_tree_s,&pLodTower->watcher)
				{
					if(pI->iId != pObj->iId)
					{
						aoiObj_tt* pWatcherObj = pTowerSpace->pSlotObj + pI->iId;
						iChanged = 0;
						if(pWatcherObj->iMode&pObj->uiMask)
						{
							iChanged = 0x1;
						}

						if(pWatcherObj->iMode&uiMask)
						{
							iChanged |= 0x2;
						}

						switch (iChanged)
						{
						case 0x1:
							{
								pTowerSpace->callback(pTowerSpace->pUserData,false,pI->iId,pObj->iId);
							}
							break;
						case 0x2:
							{
								pTowerSpace->callback(pTowerSpace->pUserData,true,pI->iId,pObj->iId);
							}
							break;
						}
					}
					
				}

				RB_FOREACH(pI,aoi_tree_s,&pLod2Tower->watcher)
				{
					if(pI->iId != pObj->iId)
					{
						aoiObj_tt* pWatcherObj = pTowerSpace->pSlotObj + pI->iId;
						iChanged = 0;
						if(pWatcherObj->iMode&pObj->uiMask)
						{
							iChanged = 0x1;
						}

						if(pWatcherObj->iMode&uiMask)
						{
							iChanged |= 0x2;
						}

						switch (iChanged)
						{
						case 0x1:
							{
								pTowerSpace->callback(pTowerSpace->pUserData,false,pI->iId,pObj->iId);
							}
							break;
						case 0x2:
							{
								pTowerSpace->callback(pTowerSpace->pUserData,true,pI->iId,pObj->iId);
							}
							break;
						}
					}
				}
			}
		}
	}

	if(pObj->iMode&WATCHER_MODE)
	{
		float bmin[3];
		float bmax[3];

		bmin[0] = pObj->last[0] - pObj->fViewRadius;
		bmin[2] = pObj->last[2] - pObj->fViewRadius;


		bmax[0] = pObj->last[0] + pObj->fViewRadius;
		bmax[2] = pObj->last[2] + pObj->fViewRadius;

		int32_t minx = 0;
		int32_t miny = 0;
		int32_t maxx = 0;
		int32_t maxy = 0;

		calcGridLoc(pTowerSpace, bmin, &minx, &miny);
		calcGridLoc(pTowerSpace, bmax, &maxx, &maxy);

		minx = minx > 0 ? minx : 0;
		miny = miny > 0 ? miny : 0;
		maxx = maxx < pTowerSpace->iMaxWidth ? maxx : pTowerSpace->iMaxWidth - 1;
		maxy = maxy < pTowerSpace->iMaxHeight ? maxy : pTowerSpace->iMaxHeight - 1;

		for (int32_t iY = miny; iY <= maxy; ++iY)
		{
			for (int32_t iX = minx; iX <= maxx; ++iX)
			{
				changeAoiObjMaskToWatcher(pTowerSpace,pObj,iX,iY,uiMask);
			}
		}
	}

	pObj->uiMask = uiMask;
}
```

## 在此AOI模式下微服务器大世界地图分割方法和传统进程分割方法的不同

**传统大世界地图分割方法**
![在这里插入图片描述](watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTYxMDI2MA==,size_16,color_FFFFFF,t_70.png)
此方法为垂直分割方法，将一个一二十公里的大地图分割成很多小地图放到各自的进程当中去处理数据，这种需要处理大世界地图边缝问题，需要做镜像数据管理，还有如果角色在一个进程中也就是某个小地图上面堆积，那么这个进程的压力会很大，别的进程却很休闲，这个也需要处理，总之很多麻烦。所以我们改成了微服务器水平分割加灯塔AOI方式

**微服务大世界地图处理方法**

![在这里插入图片描述](watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTYxMDI2MA==,size_16,color_FFFFFF,t_70-16669255366962.png)

此结构基于微服务水平分割，每个相同微服务可以并行再开很多来并行处理减少压力，比如灯塔AOI开了5个微服务我发现算力还是不够，
那么我可以在增加新的灯塔AOI微服务来并行处理数据，所以理论上一个20公里的地图撑个10多万的人不成问题。而且这种模式在服务器不需要处理无缝问题

## 一些疑问的总结

1. 为什么有了观察者集合，还需要被观察者集合

> 因为有时候想主动检查对象的状态，从怪物AI会定时检查被观察者集合的距离，决定是否发动攻击；又比如释放技能需要遍历被观察者集合，判断它们是否命中。如果没有被观察者集合，就必须遍历整个场景的对象

## 简单测试数据

![在这里插入图片描述](2.png)


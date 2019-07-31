---
layout:     post  
title:      Android定位服务  
subtitle:    
date:       2019-07-31  
author:     BY
header-img: img/post-bg-re-vs-ng2.jpg  
catalog: true  
tags: 
    - location

---

**3种PROVIDER**  
LocationManager.GPS_PROVIDER：通过gps来获取地理位置的经纬度信息，优点：获取地理位置信息精确度高，缺点：只能在户外使用，耗时，耗电。  
LocationManager.NETWORK_PROVIDER：通过移动网络的基站或者WiFi来获得地理位置，优点：只要有网络，获取速度快，耗电低，在室内室外都可以使用。  
LocationManager.PASSIVE_PROVIDER：被动的接收更新的地理位置信息，而不用自己主动请求地理位置。意思就是共享手机上其他App采集的位置信息，而不是自己主动去采集。

*FUSED_PROVIDER  
hide api, This provider combines inputs for all possible location sources to provide the best possible Location fix. It is implicitly used for all API's that involve the {@link LocationRequest} object.*

**获取位置信息, GPS搜索到的卫星颗数、并获取每颗卫星的信噪比**  
概念:信噪比，英文名称叫做SNR或S/N（SIGNAL-NOISE RATIO)，又称为讯噪比。是指一个电子设备或者电子系统中信号与噪声的比例 ,信噪比越大此颗卫星越有效（也就是说可以定位）,也就是说 设备搜索到的卫星颗数越多 设备定位效果越好，同时每颗卫星的信噪比值也要越高，如果信噪比值都是0的话；那跟没有搜索到一颗卫星效果是一样的。  
声明权限(Android M 动态申请)  

    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    
    初始化LocationManager 开启Gps
    //位置管理器
    private LocationManager manager;

    /** * 初始化定位管理 */
    private void initLocation() {
        manager = (LocationManager) getSystemService(Context.LOCATION_SERVICE);
        //判断GPS是否正常启动
        if (!manager.isProviderEnabled(LocationManager.GPS_PROVIDER)) {
            Toast.makeText(this, "请开启GPS导航", Toast.LENGTH_SHORT).show();
            //返回开启GPS导航设置界面
            Intent intent = new Intent(Settings.ACTION_LOCATION_SOURCE_SETTINGS);
            startActivityForResult(intent, 0);
            return;
        }
        //添加卫星状态改变监听
        manager.addGpsStatusListener(gpsStatusListener);
        //1000位最小的时间间隔，1为最小位移变化；也就是说每隔1000ms会回调一次位置信息
        manager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 1000, 1, mLocationListener);
    }
    private final LocationListener mLocationListener = new LocationListener() {
        @Override
        public void onLocationChanged(Location location) {
            synchronized (LocationListenerActivity.this) {
                if (mIsLocationUpdated) return;
                mIsLocationUpdated = true;
                Log.d("GpsTest", "onLocationChanged latitude:" + location.getLatitude() + " longitude:" + longitude);  //location.getLatitude 维度  location.getLongitude();经度
            }
        }

        @Override
        public void onProviderDisabled(String provider) {
        }

        @Override
        public void onProviderEnabled(String provider) {
        }

        @Override
        public void onStatusChanged(String provider, int status, Bundle extras) {
        }
    };

    private GpsStatus.Listener gpsStatusListener = new GpsStatus.Listener() {
        @Override
        public void onGpsStatusChanged(int event) {
            switch (event) {
                //卫星状态改变
                case GpsStatus.GPS_EVENT_SATELLITE_STATUS:
                    //获取当前状态
                    GpsStatus gpsStatus = manager.getGpsStatus(null);
                    //获取卫星颗数的默认最大值
                    int maxSatellites = gpsStatus.getMaxSatellites();
                    //获取所有的卫星
                    Iterator<GpsSatellite> iters = gpsStatus.getSatellites().iterator();
                    //卫星颗数统计
                    int count = 0;
                    StringBuilder sb = new StringBuilder();
                    while (iters.hasNext() && count <= maxSatellites) {
                        count++;
                        GpsSatellite s = iters.next();
                        //卫星的信噪比
                        float snr = s.getSnr();
                        sb.append("第").append(count).append("颗").append("：").append(snr).append("\n");
                    }
                    Log.d("TAG", sb.toString());
                    break;
                default:
                    break;
            }
        }
    };

当不需要相关信息时，移除监听：  

    manager.removeUpdates(mLocationListener);  
    manager.removeGpsStatusListener(gpsStatusListener);

构建自己需要的Provider  

    Criteria criteria = new Criteria();//
    criteria.setAccuracy(Criteria.ACCURACY_FINE);//设置定位精准度
    criteria.setAltitudeRequired(false);//是否要求海拔
    criteria.setBearingRequired(true);//是否要求方向
    criteria.setCostAllowed(true);//是否要求收费
    criteria.setSpeedRequired(true);//是否要求速度
    criteria.setPowerRequirement(Criteria.POWER_LOW);//设置相对省电
    criteria.setBearingAccuracy(Criteria.ACCURACY_HIGH);//设置方向精确度
    criteria.setSpeedAccuracy(Criteria.ACCURACY_HIGH);//设置速度精确度
    criteria.setHorizontalAccuracy(Criteria.ACCURACY_HIGH);//设置水平方向精确度
    criteria.setVerticalAccuracy(Criteria.ACCURACY_HIGH);//设置垂直方向精确度
    // 返回满足条件的，当前设备可用的location provider
    // 当第2个参数为false时，返回当前设备所有provider中最符合条件的那个（但是不一定可用）。
    // 当第2个参数为true时，返回当前设备所有可用的provider中最符合条件的那个。
    String rovider  = mLocationManager.getBestProvider(criteria,true);

**省电建议**  
当你调用requestLocationUpdates后，还应该是设置一个定时器，比如30s。当30s时间到了之后，就removeUpdate，不再监听地理位置了，转而使用备选的LastKnownLocation。当下次需要使用地理位置时，再重新注册监听器，监听30s，然后就移除监听器。如果对实时性要求高，我们可以在用户进入App中某个需要定位服务的场景之前，采用这个方法获取一次地理位置，把它保存下来。

如何比较Network和GPS两个Location，选出更好的那个？   
Google示例代码  

    private static final int TWO_MINUTES = 1000 * 60 * 2;
    /** Determines whether one Location reading is better than the current Location fix
     * @param location  The new Location that you want to evaluate
     * @param currentBestLocation  The current Location fix, to which you want to compare the new one
     */
    protected boolean isBetterLocation(Location location, Location currentBestLocation) {
        if (currentBestLocation == null) {
            // A new location is always better than no location
            return true;
        }
        // Check whether the new location fix is newer or older
        long timeDelta = location.getTime() - currentBestLocation.getTime();
        boolean isSignificantlyNewer = timeDelta > TWO_MINUTES;
        boolean isSignificantlyOlder = timeDelta < -TWO_MINUTES;
        boolean isNewer = timeDelta > 0;
        // If it's been more than two minutes since the current location, use the new location
        // because the user has likely moved
        if (isSignificantlyNewer) {
            return true;
            // If the new location is more than two minutes older, it must be worse
        } else if (isSignificantlyOlder) {
            return false;
        }
        // Check whether the new location fix is more or less accurate
        int accuracyDelta = (int) (location.getAccuracy() - currentBestLocation.getAccuracy());
        boolean isLessAccurate = accuracyDelta > 0;
        boolean isMoreAccurate = accuracyDelta < 0;
        boolean isSignificantlyLessAccurate = accuracyDelta > 200;
        // Check if the old and new location are from the same provider
        boolean isFromSameProvider = isSameProvider(location.getProvider(),
                currentBestLocation.getProvider());
        // Determine location quality using a combination of timeliness and accuracy
        if (isMoreAccurate) {
            return true;
        } else if (isNewer && !isLessAccurate) {
            return true;
        } else if (isNewer && !isSignificantlyLessAccurate && isFromSameProvider) {
            return true;
        }
        return false;
    }
    /** Checks whether two providers are the same */
    private boolean isSameProvider(String provider1, String provider2) {
        if (provider1 == null) {
            return provider2 == null;
        }
        return provider1.equals(provider2);
    }



获取LastknownPostion   

    Location gpsLocation = null;
    gpsLocation = locationManager.getLastKnownLocation(LocationManager.GPS_PROVIDER);


Geofence  


了解腾讯定位SDK和高德定位SDK接入逻辑


sample  
https://github.com/googlesamples/android-play-location

参考资料   
http://unclechen.github.io/2016/09/02/Android%E5%9C%B0%E7%90%86%E4%BD%8D%E7%BD%AE%E6%9C%8D%E5%8A%A1%E8%A7%A3%E6%9E%90/
https://developer.android.com/training/location
https://github.com/baurine/android-location-study/blob/master/note/location-note.md
name: Build Feishu Alarm APK

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Create project files
        run: |
          mkdir -p app/src/main/java/com/example/feishualarm

          cat > settings.gradle <<'EOF'
          pluginManagement {
              repositories {
                  google()
                  mavenCentral()
                  gradlePluginPortal()
              }
          }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
              repositories {
                  google()
                  mavenCentral()
              }
          }
          rootProject.name = "FeishuAlarm"
          include ':app'
          EOF

          cat > build.gradle <<'EOF'
          plugins {
              id 'com.android.application' version '8.5.2' apply false
          }
          EOF

          cat > gradle.properties <<'EOF'
          org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
          android.builder.sdkDownload=true
          EOF

          cat > app/build.gradle <<'EOF'
          plugins {
              id 'com.android.application'
          }

          android {
              namespace 'com.example.feishualarm'
              compileSdk 34

              defaultConfig {
                  applicationId "com.example.feishualarm"
                  minSdk 27
                  targetSdk 34
                  versionCode 1
                  versionName "1.0"
              }
          }
          EOF

          cat > app/src/main/AndroidManifest.xml <<'EOF'
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">

              <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
              <uses-permission android:name="android.permission.WAKE_LOCK" />
              <uses-permission android:name="android.permission.VIBRATE" />
              <uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />
              <uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" />
              <uses-permission android:name="android.permission.USE_FULL_SCREEN_INTENT" />
              <uses-permission android:name="android.permission.ACCESS_NOTIFICATION_POLICY" />

              <application
                  android:allowBackup="true"
                  android:label="飞书夜班铃声"
                  android:theme="@android:style/Theme.Material.NoActionBar">

                  <activity
                      android:name=".MainActivity"
                      android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>

                  <activity
                      android:name=".MainActivity$AlarmActivity"
                      android:exported="false"
                      android:launchMode="singleInstance"
                      android:excludeFromRecents="true"
                      android:showWhenLocked="true"
                      android:turnScreenOn="true" />

                  <service
                      android:name=".MainActivity$AlarmService"
                      android:exported="false"
                      android:foregroundServiceType="specialUse">
                      <property
                          android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
                          android:value="feishu alarm guard" />
                  </service>

                  <service
                      android:name=".MainActivity$FeishuNotificationListener"
                      android:exported="false"
                      android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE">
                      <intent-filter>
                          <action android:name="android.service.notification.NotificationListenerService" />
                      </intent-filter>
                  </service>
              </application>
          </manifest>
          EOF

          cat > app/src/main/java/com/example/feishualarm/MainActivity.java <<'EOF'
          package com.example.feishualarm;

          import android.app.Activity;
          import android.app.AlertDialog;
          import android.app.KeyguardManager;
          import android.app.Notification;
          import android.app.NotificationChannel;
          import android.app.NotificationManager;
          import android.app.PendingIntent;
          import android.app.Service;
          import android.content.Context;
          import android.content.Intent;
          import android.content.SharedPreferences;
          import android.content.pm.PackageManager;
          import android.database.Cursor;
          import android.graphics.Color;
          import android.media.AudioAttributes;
          import android.media.AudioManager;
          import android.media.MediaPlayer;
          import android.media.Ringtone;
          import android.media.RingtoneManager;
          import android.net.Uri;
          import android.os.Build;
          import android.os.Bundle;
          import android.os.IBinder;
          import android.os.PowerManager;
          import android.os.VibrationEffect;
          import android.os.Vibrator;
          import android.provider.Settings;
          import android.view.Gravity;
          import android.view.View;
          import android.view.WindowManager;
          import android.widget.AdapterView;
          import android.widget.ArrayAdapter;
          import android.widget.Button;
          import android.widget.CheckBox;
          import android.widget.LinearLayout;
          import android.widget.ScrollView;
          import android.widget.Spinner;
          import android.widget.TextView;
          import android.widget.TimePicker;
          import android.widget.Toast;
          import org.json.JSONArray;
          import org.json.JSONObject;
          import java.io.File;
          import java.io.FileOutputStream;
          import java.io.InputStream;
          import java.io.OutputStream;
          import java.util.ArrayList;
          import java.util.Calendar;
          import java.util.List;

          public class MainActivity extends Activity {

              private TextView status;
              private LinearLayout scheduleList;
              private Spinner ringtoneSpinner;
              private List<RingtoneHelper.Option> builtIn;
              private TextView customLabel;
              private CheckBox enableBox;

              @Override
              protected void onCreate(Bundle savedInstanceState) {
                  super.onCreate(savedInstanceState);
                  builtIn = RingtoneHelper.getBuiltIn(this);
                  buildUi();
                  requestNotificationPermission();
              }

              private void buildUi() {
                  ScrollView scroll = new ScrollView(this);
                  LinearLayout root = new LinearLayout(this);
                  root.setOrientation(LinearLayout.VERTICAL);
                  root.setPadding(dp(this, 20), dp(this, 20), dp(this, 20), dp(this, 30));
                  root.setBackgroundColor(0xFF0D0D12);
                  scroll.addView(root);

                  TextView title = new TextView(this);
                  title.setText("飞书夜班铃声");
                  title.setTextColor(Color.WHITE);
                  title.setTextSize(26);
                  title.setGravity(Gravity.CENTER);
                  root.addView(title);

                  status = new TextView(this);
                  status.setTextColor(0xFFB0B0C0);
                  status.setTextSize(13);
                  status.setPadding(0, dp(this, 8), 0, dp(this, 16));
                  root.addView(status);

                  enableBox = new CheckBox(this);
                  enableBox.setText("启用飞书响铃");
                  enableBox.setTextColor(Color.WHITE);
                  enableBox.setTextSize(16);
                  enableBox.setChecked(ScheduleStore.isEnabled(this));
                  enableBox.setOnCheckedChangeListener((v, checked) -> {
                      ScheduleStore.setEnabled(this, checked);
                      if (checked) {
                          AlarmService.startGuard(this);
                      } else {
                          AlarmService.stopGuard(this);
                      }
                  });
                  root.addView(enableBox);

                  root.addView(section("权限"));
                  root.addView(button("通知使用权", v ->
                          startActivity(new Intent(Settings.ACTION_NOTIFICATION_LISTENER_SETTINGS))));
                  if (Build.VERSION.SDK_INT >= 34) {
                      root.addView(button("全屏通知权限", v -> {
                          try {
                              startActivity(new Intent(
                                      Settings.ACTION_MANAGE_APP_USE_FULL_SCREEN_INTENT,
                                      Uri.parse("package:" + getPackageName())));
                          } catch (Exception e) {
                              toast("请到系统设置里允许全屏通知");
                          }
                      }));
                  }
                  root.addView(button("电池优化白名单", v -> {
                      try {
                          startActivity(new Intent(
                                  Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS,
                                  Uri.parse("package:" + getPackageName())));
                      } catch (Exception e) {
                          toast("请到系统设置里关闭电池优化");
                      }
                  }));
                  root.addView(button("勿扰访问权限", v ->
                          startActivity(new Intent(Settings.ACTION_NOTIFICATION_POLICY_ACCESS_SETTINGS))));

                  root.addView(section("铃声"));
                  ringtoneSpinner = new Spinner(this);
                  ArrayAdapter<String> adapter = new ArrayAdapter<>(
                          this, android.R.layout.simple_spinner_item);
                  for (RingtoneHelper.Option option : builtIn) {
                      adapter.add(option.title);
                  }
                  adapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
                  ringtoneSpinner.setAdapter(adapter);

                  String saved = ScheduleStore.prefs(this).getString(ScheduleStore.KEY_RINGTONE, "");
                  int selected = 0;
                  for (int i = 0; i < builtIn.size(); i++) {
                      if (("builtin:" + builtIn.get(i).uri).equals(saved)) {
                          selected = i;
                      }
                  }
                  ringtoneSpinner.setSelection(selected);
                  ringtoneSpinner.setOnItemSelectedListener(new AdapterView.OnItemSelectedListener() {
                      @Override
                      public void onItemSelected(AdapterView<?> parent, View view, int position, long id) {
                          RingtoneHelper.setSelected(MainActivity.this,
                                  "builtin:" + builtIn.get(position).uri);
                          updateRingtoneLabel();
                      }

                      @Override
                      public void onNothingSelected(AdapterView<?> parent) {
                      }
                  });
                  root.addView(ringtoneSpinner);

                  customLabel = new TextView(this);
                  customLabel.setTextColor(0xFF9090A0);
                  customLabel.setTextSize(12);
                  customLabel.setPadding(0, dp(this, 6), 0, dp(this, 8));
                  root.addView(customLabel);

                  LinearLayout ringRow = new LinearLayout(this);
                  ringRow.setOrientation(LinearLayout.HORIZONTAL);
                  Button preview = button("试听", v -> RingtoneHelper.playPreview(MainActivity.this));
                  Button pick = button("选择自定义铃声", v -> pickCustomRingtone());
                  ringRow.addView(preview, new LinearLayout.LayoutParams(0, dp(this, 48), 1));
                  ringRow.addView(pick, new LinearLayout.LayoutParams(0, dp(this, 48), 1));
                  root.addView(ringRow);

                  root.addView(section("启用时间段"));
                  scheduleList = new LinearLayout(this);
                  scheduleList.setOrientation(LinearLayout.VERTICAL);
                  root.addView(scheduleList);
                  root.addView(button("添加时间段", v -> showAddScheduleDialog()));
                  renderSchedules();

                  root.addView(section("测试"));
                  root.addView(button("测试响铃", v ->
                          AlarmService.trigger(this, "测试消息", "这是测试铃声")));

                  TextView note = new TextView(this);
                  note.setText("说明：需要允许通知使用权；飞书不能被静音；手机重启后请打开一次本 App。安卓不允许任何 App 阻止强制停止、重启或音量键。");
                  note.setTextColor(0xFF808090);
                  note.setTextSize(12);
                  note.setPadding(0, dp(this, 20), 0, 0);
                  root.addView(note);

                  setContentView(scroll);
                  updateRingtoneLabel();
              }

              private TextView section(String text) {
                  TextView view = new TextView(this);
                  view.setText(text);
                  view.setTextColor(0xFF7CC7FF);
                  view.setTextSize(13);
                  view.setPadding(0, dp(this, 24), 0, dp(this, 8));
                  return view;
              }

              private Button button(String text, View.OnClickListener listener) {
                  Button button = new Button(this);
                  button.setText(text);
                  button.setAllCaps(false);
                  button.setTextSize(15);
                  button.setOnClickListener(listener);
                  button.setLayoutParams(new LinearLayout.LayoutParams(
                          LinearLayout.LayoutParams.MATCH_PARENT, dp(this, 48)));
                  return button;
              }

              private void renderSchedules() {
                  if (scheduleList == null) return;
                  scheduleList.removeAllViews();
                  List<ScheduleStore.Schedule> schedules = ScheduleStore.getSchedules(this);
                  if (schedules.isEmpty()) {
                      TextView empty = new TextView(this);
                      empty.setText("还没有时间段，点下面添加");
                      empty.setTextColor(0xFF808090);
                      empty.setTextSize(13);
                      scheduleList.addView(empty);
                      return;
                  }
                  for (int i = 0; i < schedules.size(); i++) {
                      final int index = i;
                      LinearLayout row = new LinearLayout(this);
                      row.setOrientation(LinearLayout.HORIZONTAL);
                      row.setGravity(Gravity.CENTER_VERTICAL);

                      TextView text = new TextView(this);
                      text.setText(ScheduleStore.describe(schedules.get(i)));
                      text.setTextColor(Color.WHITE);
                      text.setTextSize(13);
                      text.setLayoutParams(new LinearLayout.LayoutParams(
                              0, LinearLayout.LayoutParams.WRAP_CONTENT, 1));

                      Button delete = new Button(this);
                      delete.setText("删除");
                      delete.setAllCaps(false);
                      delete.setOnClickListener(v -> {
                          List<ScheduleStore.Schedule> list = ScheduleStore.getSchedules(MainActivity.this);
                          if (index < list.size()) list.remove(index);
                          ScheduleStore.saveSchedules(MainActivity.this, list);
                          renderSchedules();
                      });

                      row.addView(text);
                      row.addView(delete);
                      scheduleList.addView(row);
                  }
              }

              private void showAddScheduleDialog() {
                  LinearLayout box = new LinearLayout(this);
                  box.setOrientation(LinearLayout.VERTICAL);
                  box.setPadding(dp(this, 16), dp(this, 8), dp(this, 16), dp(this, 8));

                  TextView dayLabel = new TextView(this);
                  dayLabel.setText("选择生效的星期：");
                  dayLabel.setTextColor(Color.DKGRAY);
                  box.addView(dayLabel);

                  String[] names = {"周一", "周二", "周三", "周四", "周五", "周六", "周日"};
                  CheckBox[] boxes = new CheckBox[7];
                  for (int i = 0; i < 7; i++) {
                      boxes[i] = new CheckBox(this);
                      boxes[i].setText(names[i]);
                      box.addView(boxes[i]);
                  }

                  TextView startLabel = new TextView(this);
                  startLabel.setText("开始时间：");
                  startLabel.setTextColor(Color.DKGRAY);
                  startLabel.setPadding(0, dp(this, 10), 0, 0);
                  box.addView(startLabel);
                  TimePicker start = new TimePicker(this);
                  start.setIs24HourView(true);
                  box.addView(start);

                  TextView endLabel = new TextView(this);
                  endLabel.setText("结束时间（可跨天）：");
                  endLabel.setTextColor(Color.DKGRAY);
                  endLabel.setPadding(0, dp(this, 10), 0, 0);
                  box.addView(endLabel);
                  TimePicker end = new TimePicker(this);
                  end.setIs24HourView(true);
                  box.addView(end);

                  ScrollView scroll = new ScrollView(this);
                  scroll.addView(box);

                  new AlertDialog.Builder(this)
                          .setTitle("添加时间段")
                          .setView(scroll)
                          .setPositiveButton("添加", (dialog, which) -> {
                              int days = 0;
                              for (int i = 0; i < 7; i++) {
                                  if (boxes[i].isChecked()) days |= (1 << i);
                              }
                              if (days == 0) {
                                  toast("至少选一天");
                                  return;
                              }
                              int startMinutes = start.getHour() * 60 + start.getMinute();
                              int endMinutes = end.getHour() * 60 + end.getMinute();
                              List<ScheduleStore.Schedule> list = ScheduleStore.getSchedules(MainActivity.this);
                              list.add(new ScheduleStore.Schedule(days, startMinutes, endMinutes));
                              ScheduleStore.saveSchedules(MainActivity.this, list);
                              renderSchedules();
                          })
                          .setNegativeButton("取消", null)
                          .show();
              }

              private void pickCustomRingtone() {
                  Intent intent = new Intent(Intent.ACTION_OPEN_DOCUMENT);
                  intent.addCategory(Intent.CATEGORY_OPENABLE);
                  intent.setType("audio/*");
                  startActivityForResult(intent, 100);
              }

              @Override
              protected void onActivityResult(int requestCode, int resultCode, Intent data) {
                  super.onActivityResult(requestCode, resultCode, data);
                  if (requestCode == 100 && resultCode == RESULT_OK && data != null && data.getData() != null) {
                      try {
                          InputStream in = getContentResolver().openInputStream(data.getData());
                          File output = new File(getFilesDir(), "custom_ringtone");
                          OutputStream out = new FileOutputStream(output);
                          byte[] buffer = new byte[8192];
                          int length;
                          while (in != null && (length = in.read(buffer)) > 0) {
                              out.write(buffer, 0, length);
                          }
                          if (in != null) in.close();
                          out.close();
                          RingtoneHelper.setSelected(this, "custom:" + output.getAbsolutePath());
                          updateRingtoneLabel();
                          toast("自定义铃声已保存");
                      } catch (Exception e) {
                          toast("保存失败：" + e.getMessage());
                      }
                  }
              }

              private void updateRingtoneLabel() {
                  if (customLabel == null) return;
                  String value = ScheduleStore.prefs(this).getString(ScheduleStore.KEY_RINGTONE, "");
                  if (value.startsWith("custom:")) {
                      customLabel.setText("当前：自定义铃声");
                  } else {
                      customLabel.setText("当前：系统铃声");
                  }
              }

              private void updateStatus() {
                  String notification = hasNotificationAccess() ? "已开启" : "未开启";
                  String battery = isIgnoringBattery() ? "已忽略" : "未忽略";
                  String dnd = isDndAllowed() ? "已授权" : "未授权";
                  String fullScreen;
                  if (Build.VERSION.SDK_INT >= 34) {
                      NotificationManager nm = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
                      fullScreen = nm.canUseFullScreenIntent() ? "已允许" : "未允许";
                  } else {
                      fullScreen = "系统版本无需设置";
                  }
                  status.setText("通知使用权：" + notification
                          + "\n全屏通知：" + fullScreen
                          + "\n电池优化：" + battery
                          + "\n勿扰权限：" + dnd);
              }

              private boolean hasNotificationAccess() {
                  String enabled = Settings.Secure.getString(
                          getContentResolver(), "enabled_notification_listeners");
                  return enabled != null && enabled.contains(getPackageName());
              }

              private boolean isIgnoringBattery() {
                  PowerManager pm = (PowerManager) getSystemService(POWER_SERVICE);
                  return pm.isIgnoringBatteryOptimizations(getPackageName());
              }

              private boolean isDndAllowed() {
                  NotificationManager nm = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
                  return nm.isNotificationPolicyAccessGranted();
              }

              private void requestNotificationPermission() {
                  if (Build.VERSION.SDK_INT >= 33 &&
                          checkSelfPermission("android.permission.POST_NOTIFICATIONS")
                                  != PackageManager.PERMISSION_GRANTED) {
                      requestPermissions(new String[]{"android.permission.POST_NOTIFICATIONS"}, 1);
                  }
              }

              @Override
              protected void onResume() {
                  super.onResume();
                  updateStatus();
                  if (ScheduleStore.isEnabled(this)) {
                      AlarmService.startGuard(this);
                  }
              }

              private void toast(String text) {
                  Toast.makeText(this, text, Toast.LENGTH_SHORT).show();
              }

              static int dp(Context context, int value) {
                  return Math.round(value * context.getResources().getDisplayMetrics().density);
              }

              public static class ScheduleStore {
                  public static final String PREFS = "feishu_alarm";
                  public static final String KEY_SCHEDULES = "schedules";
                  public static final String KEY_ENABLED = "enabled";
                  public static final String KEY_RINGTONE = "ringtone";
                  public static final String KEY_LAST_RING = "last_ring";

                  public static class Schedule {
                      public final int days;
                      public final int start;
                      public final int end;

                      public Schedule(int days, int start, int end) {
                          this.days = days;
                          this.start = start;
                          this.end = end;
                      }
                  }

                  public static SharedPreferences prefs(Context context) {
                      return context.getSharedPreferences(PREFS, Context.MODE_PRIVATE);
                  }

                  public static boolean isEnabled(Context context) {
                      return prefs(context).getBoolean(KEY_ENABLED, true);
                  }

                  public static void setEnabled(Context context, boolean enabled) {
                      prefs(context).edit().putBoolean(KEY_ENABLED, enabled).apply();
                  }

                  public static List<Schedule> getSchedules(Context context) {
                      List<Schedule> result = new ArrayList<>();
                      try {
                          JSONArray array = new JSONArray(
                                  prefs(context).getString(KEY_SCHEDULES, "[]"));
                          for (int i = 0; i < array.length(); i++) {
                              JSONObject object = array.getJSONObject(i);
                              result.add(new Schedule(
                                      object.optInt("days"),
                                      object.optInt("start"),
                                      object.optInt("end")));
                          }
                      } catch (Exception ignored) {
                      }
                      return result;
                  }

                  public static void saveSchedules(Context context, List<Schedule> schedules) {
                      JSONArray array = new JSONArray();
                      for (Schedule schedule : schedules) {
                          JSONObject object = new JSONObject();
                          try {
                              object.put("days", schedule.days);
                              object.put("start", schedule.start);
                              object.put("end", schedule.end);
                          } catch (Exception ignored) {
                          }
                          array.put(object);
                      }
                      prefs(context).edit().putString(KEY_SCHEDULES, array.toString()).apply();
                  }

                  public static boolean isActive(Context context, long now) {
                      if (!isEnabled(context)) return false;
                      List<Schedule> schedules = getSchedules(context);
                      if (schedules.isEmpty()) return false;

                      Calendar calendar = Calendar.getInstance();
                      calendar.setTimeInMillis(now);
                      int dayOfWeek = calendar.get(Calendar.DAY_OF_WEEK);
                      int todayBit = 1 << ((dayOfWeek + 5) % 7);
                      int yesterdayBit = 1 << ((dayOfWeek + 4) % 7);
                      int minutes = calendar.get(Calendar.HOUR_OF_DAY) * 60
                              + calendar.get(Calendar.MINUTE);

                      for (Schedule schedule : schedules) {
                          if (schedule.start == schedule.end) {
                              if ((schedule.days & todayBit) != 0) return true;
                          } else if (schedule.start < schedule.end) {
                              if ((schedule.days & todayBit) != 0
                                      && minutes >= schedule.start
                                      && minutes < schedule.end) {
                                  return true;
                              }
                          } else {
                              if (((schedule.days & todayBit) != 0 && minutes >= schedule.start)
                                      || ((schedule.days & yesterdayBit) != 0 && minutes < schedule.end)) {
                                  return true;
                              }
                          }
                      }
                      return false;
                  }

                  public static String describe(Schedule schedule) {
                      String[] names = {"周一", "周二", "周三", "周四", "周五", "周六", "周日"};
                      StringBuilder builder = new StringBuilder();
                      for (int i = 0; i < 7; i++) {
                          if ((schedule.days & (1 << i)) != 0) {
                              if (builder.length() > 0) builder.append("、");
                              builder.append(names[i]);
                          }
                      }
                      if (builder.length() == 0) builder.append("未选日期");
                      return builder.toString() + "  " + time(schedule.start) + " - " + time(schedule.end);
                  }

                  public static String time(int minutes) {
                      return String.format("%02d:%02d", minutes / 60, minutes % 60);
                  }
              }

              public static class RingtoneHelper {
                  public static class Option {
                      public final String title;
                      public final Uri uri;

                      public Option(String title, Uri uri) {
                          this.title = title;
                          this.uri = uri;
                      }
                  }

                  private static Ringtone preview;

                  public static List<Option> getBuiltIn(Context context) {
                      List<Option> result = new ArrayList<>();
                      RingtoneManager manager = new RingtoneManager(context);
                      manager.setType(RingtoneManager.TYPE_ALARM);
                      Cursor cursor = manager.getCursor();
                      while (cursor != null && cursor.moveToNext() && result.size() < 5) {
                          Uri uri = manager.getRingtoneUri(cursor.getPosition());
                          if (uri != null) {
                              result.add(new Option(
                                      cursor.getString(RingtoneManager.TITLE_COLUMN_INDEX), uri));
                          }
                      }
                      if (cursor != null) cursor.close();
                      if (result.isEmpty()) {
                          result.add(new Option("默认闹钟",
                                  RingtoneManager.getDefaultUri(RingtoneManager.TYPE_ALARM)));
                      }
                      return result;
                  }

                  public static void setSelected(Context context, String value) {
                      ScheduleStore.prefs(context).edit()
                              .putString(ScheduleStore.KEY_RINGTONE, value).apply();
                  }

                  public static Uri getSelectedUri(Context context) {
                      String value = ScheduleStore.prefs(context)
                              .getString(ScheduleStore.KEY_RINGTONE, "");
                      if (value.startsWith("custom:")) {
                          File file = new File(value.substring(7));
                          if (file.exists()) return Uri.fromFile(file);
                      }
                      if (value.startsWith("builtin:")) {
                          try {
                              return Uri.parse(value.substring(8));
                          } catch (Exception ignored) {
                          }
                      }
                      return RingtoneManager.getDefaultUri(RingtoneManager.TYPE_ALARM);
                  }

                  public static void playPreview(Context context) {
                      try {
                          if (preview != null) preview.stop();
                          preview = RingtoneManager.getRingtone(context, getSelectedUri(context));
                          if (preview != null) preview.play();
                      } catch (Exception ignored) {
                      }
                  }
              }

              public static class FeishuNotificationListener extends android.service.notification.NotificationListenerService {
                  private static final String[] PACKAGES = {
                          "com.ss.android.lark",
                          "com.larksuite.suite"
                  };

                  @Override
                  public void onNotificationPosted(android.service.notification.StatusBarNotification sbn) {
                      if (sbn == null) return;
                      boolean isFeishu = false;
                      for (String pkg : PACKAGES) {
                          if (pkg.equals(sbn.getPackageName())) {
                              isFeishu = true;
                              break;
                          }
                      }
                      if (!isFeishu) return;
                      if (!ScheduleStore.isEnabled(this)) return;
                      if (!ScheduleStore.isActive(this, System.currentTimeMillis())) return;
                      if (AlarmService.isRinging) return;

                      long now = System.currentTimeMillis();
                      long last = ScheduleStore.prefs(this).getLong(ScheduleStore.KEY_LAST_RING, 0);
                      if (now - last < 15000) return;
                      ScheduleStore.prefs(this).edit().putLong(ScheduleStore.KEY_LAST_RING, now).apply();

                      Notification notification = sbn.getNotification();
                      Bundle extras = notification.extras;
                      String title = extras.getString(Notification.EXTRA_TITLE, "飞书消息");
                      String text = extras.getString(Notification.EXTRA_TEXT, "");
                      if (text == null || text.isEmpty()) {
                          text = extras.getString(Notification.EXTRA_BIG_TEXT, "收到一条飞书消息");
                      }
                      AlarmService.trigger(this, title, text);
                  }
              }

              public static class AlarmService extends Service {
                  public static final String ACTION_TRIGGER = "com.example.feishualarm.TRIGGER";
                  public static final String ACTION_STOP = "com.example.feishualarm.STOP";
                  private static final String CHANNEL_GUARD = "feishu_guard";
                  private static final String CHANNEL_ALARM = "feishu_alarm";
                  private static final int ID_GUARD = 1;
                  private static final int ID_ALARM = 2;

                  public static volatile boolean isRinging = false;

                  private MediaPlayer player;
                  private Vibrator vibrator;
                  private PowerManager.WakeLock wakeLock;
                  private AudioManager audioManager;
                  private int originalVolume = -1;
                  private String title = "飞书消息";
                  private String text = "";

                  public static void startGuard(Context context) {
                      Intent intent = new Intent(context, AlarmService.class);
                      if (Build.VERSION.SDK_INT >= 26) {
                          context.startForegroundService(intent);
                      } else {
                          context.startService(intent);
                      }
                  }

                  public static void stopGuard(Context context) {
                      context.stopService(new Intent(context, AlarmService.class));
                  }

                  public static void trigger(Context context, String title, String text) {
                      Intent intent = new Intent(context, AlarmService.class);
                      intent.setAction(ACTION_TRIGGER);
                      intent.putExtra("title", title);
                      intent.putExtra("text", text);
                      if (Build.VERSION.SDK_INT >= 26) {
                          context.startForegroundService(intent);
                      } else {
                          context.startService(intent);
                      }
                  }

                  public static void stop(Context context) {
                      Intent intent = new Intent(context, AlarmService.class);
                      intent.setAction(ACTION_STOP);
                      context.startService(intent);
                  }

                  @Override
                  public void onCreate() {
                      super.onCreate();
                      createChannels();
                      startForeground(ID_GUARD, guardNotification());
                  }

                  @Override
                  public int onStartCommand(Intent intent, int flags, int startId) {
                      if (intent != null && ACTION_TRIGGER.equals(intent.getAction())) {
                          String newTitle = intent.getStringExtra("title");
                          String newText = intent.getStringExtra("text");
                          if (newTitle != null) title = newTitle;
                          if (newText != null) text = newText;
                          triggerAlarm();
                      } else if (intent != null && ACTION_STOP.equals(intent.getAction())) {
                          stopAlarm();
                      }
                      return START_STICKY;
                  }

                  private void triggerAlarm() {
                      if (isRinging) return;
                      isRinging = true;
                      startSoundAndVibration();
                      NotificationManager nm = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
                      nm.notify(ID_ALARM, alarmNotification());
                      launchAlarmActivity();
                  }

                  private void stopAlarm() {
                      isRinging = false;
                      stopSoundAndVibration();
                      NotificationManager nm = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
                      nm.cancel(ID_ALARM);
                      nm.notify(ID_GUARD, guardNotification());
                  }

                  private void startSoundAndVibration() {
                      try {
                          audioManager = (AudioManager) getSystemService(AUDIO_SERVICE);
                          originalVolume = audioManager.getStreamVolume(AudioManager.STREAM_ALARM);
                          audioManager.setStreamVolume(AudioManager.STREAM_ALARM,
                                  audioManager.getStreamMaxVolume(AudioManager.STREAM_ALARM), 0);
                      } catch (Exception ignored) {
                      }

                      try {
                          player = new MediaPlayer();
                          player.setAudioAttributes(new AudioAttributes.Builder()
                                  .setUsage(AudioAttributes.USAGE_ALARM)
                                  .setContentType(AudioAttributes.CONTENT_TYPE_SONIFICATION)
                                  .build());
                          player.setDataSource(this, RingtoneHelper.getSelectedUri(this));
                          player.setLooping(true);
                          player.prepare();
                          player.start();
                      } catch (Exception e) {
                          try {
                              player = MediaPlayer.create(this,
                                      RingtoneManager.getDefaultUri(RingtoneManager.TYPE_ALARM));
                              if (player != null) {
                                  player.setLooping(true);
                                  player.start();
                              }
                          } catch (Exception ignored) {
                          }
                      }

                      try {
                          vibrator = (Vibrator) getSystemService(VIBRATOR_SERVICE);
                          if (vibrator != null && vibrator.hasVibrator()) {
                              long[] pattern = {0, 800, 400, 800, 400};
                              if (Build.VERSION.SDK_INT >= 26) {
                                  vibrator.vibrate(VibrationEffect.createWaveform(pattern, 0));
                              } else {
                                  vibrator.vibrate(pattern, 0);
                              }
                          }
                      } catch (Exception ignored) {
                      }

                      PowerManager pm = (PowerManager) getSystemService(POWER_SERVICE);
                      wakeLock = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "FeishuAlarm::Ring");
                      wakeLock.acquire();
                  }

                  private void stopSoundAndVibration() {
                      if (player != null) {
                          try {
                              player.stop();
                          } catch (Exception ignored) {
                          }
                          player.release();
                          player = null;
                      }
                      if (vibrator != null) {
                          vibrator.cancel();
                          vibrator = null;
                      }
                      if (wakeLock != null && wakeLock.isHeld()) {
                          wakeLock.release();
                          wakeLock = null;
                      }
                      if (audioManager != null && originalVolume >= 0) {
                          try {
                              audioManager.setStreamVolume(
                                      AudioManager.STREAM_ALARM, originalVolume, 0);
                          } catch (Exception ignored) {
                          }
                          originalVolume = -1;
                      }
                  }

                  private void launchAlarmActivity() {
                      Intent intent = new Intent(this, AlarmActivity.class);
                      intent.putExtra("title", title);
                      intent.putExtra("text", text);
                      intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                              | Intent.FLAG_ACTIVITY_CLEAR_TOP
                              | Intent.FLAG_ACTIVITY_SINGLE_TOP);
                      try {
                          startActivity(intent);
                      } catch (Exception ignored) {
                      }
                  }

                  private void createChannels() {
                      NotificationManager nm = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
                      NotificationChannel guard = new NotificationChannel(
                              CHANNEL_GUARD, "飞书铃声守护", NotificationManager.IMPORTANCE_LOW);
                      nm.createNotificationChannel(guard);

                      NotificationChannel alarm = new NotificationChannel(
                              CHANNEL_ALARM, "飞书响铃", NotificationManager.IMPORTANCE_HIGH);
                      alarm.setSound(null, null);
                      alarm.enableVibration(false);
                      nm.createNotificationChannel(alarm);
                  }

                  private Notification guardNotification() {
                      Intent open = new Intent(this, MainActivity.class);
                      PendingIntent pi = PendingIntent.getActivity(this, 10, open,
                              PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE);
                      return new Notification.Builder(this, CHANNEL_GUARD)
                              .setSmallIcon(android.R.drawable.ic_lock_idle_alarm)
                              .setContentTitle("飞书夜班铃声")
                              .setContentText("正在监听飞书消息")
                              .setOngoing(true)
                              .setContentIntent(pi)
                              .build();
                  }

                  private Notification alarmNotification() {
                      Intent open = new Intent(this, AlarmActivity.class);
                      open.putExtra("title", title);
                      open.putExtra("text", text);
                      PendingIntent pi = PendingIntent.getActivity(this, 11, open,
                              PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE);
                      return new Notification.Builder(this, CHANNEL_ALARM)
                              .setSmallIcon(android.R.drawable.ic_lock_idle_alarm)
                              .setContentTitle("飞书消息")
                              .setContentText(title)
                              .setCategory(Notification.CATEGORY_ALARM)
                              .setPriority(Notification.PRIORITY_MAX)
                              .setVisibility(Notification.VISIBILITY_PUBLIC)
                              .setOngoing(true)
                              .setContentIntent(pi)
                              .setFullScreenIntent(pi, true)
                              .build();
                  }

                  @Override
                  public void onDestroy() {
                      stopAlarm();
                      super.onDestroy();
                  }

                  @Override
                  public IBinder onBind(Intent intent) {
                      return null;
                  }
              }

              public static class AlarmActivity extends Activity {

                  @Override
                  protected void onCreate(Bundle savedInstanceState) {
                      super.onCreate(savedInstanceState);
                      if (Build.VERSION.SDK_INT >= 27) {
                          setShowWhenLocked(true);
                          setTurnScreenOn(true);
                      }
                      getWindow().addFlags(
                              WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
                                      | WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED
                                      | WindowManager.LayoutParams.FLAG_TURN_SCREEN_ON
                                      | WindowManager.LayoutParams.FLAG_DISMISS_KEYGUARD);

                      String title = getIntent().getStringExtra("title");
                      String text = getIntent().getStringExtra("text");
                      if (title == null) title = "飞书消息";
                      if (text == null) text = "";

                      LinearLayout root = new LinearLayout(this);
                      root.setOrientation(LinearLayout.VERTICAL);
                      root.setGravity(Gravity.CENTER);
                      root.setPadding(dp(this, 24), dp(this, 24), dp(this, 24), dp(this, 24));
                      root.setBackgroundColor(0xFF1A0B0B);

                      TextView label = new TextView(this);
                      label.setText("飞书消息");
                      label.setTextColor(0xFFFF6B6B);
                      label.setTextSize(18);
                      label.setGravity(Gravity.CENTER);
                      root.addView(label);

                      TextView titleView = new TextView(this);
                      titleView.setText(title);
                      titleView.setTextColor(Color.WHITE);
                      titleView.setTextSize(26);
                      titleView.setGravity(Gravity.CENTER);
                      titleView.setPadding(0, dp(this, 20), 0, dp(this, 10));
                      root.addView(titleView);

                      TextView textView = new TextView(this);
                      textView.setText(text);
                      textView.setTextColor(0xFFE0E0E0);
                      textView.setTextSize(16);
                      textView.setGravity(Gravity.CENTER);
                      textView.setPadding(0, 0, 0, dp(this, 30));
                      root.addView(textView);

                      TextView note = new TextView(this);
                      note.setText("必须解锁手机，进入本 App 点击停止");
                      note.setTextColor(0xFFFFB3B3);
                      note.setTextSize(13);
                      note.setGravity(Gravity.CENTER);
                      root.addView(note);

                      Button stop = new Button(this);
                      stop.setText("停止铃声（需要解锁）");
                      stop.setTextSize(16);
                      stop.setAllCaps(false);
                      stop.setOnClickListener(v -> requestStop());
                      root.addView(stop, new LinearLayout.LayoutParams(
                              LinearLayout.LayoutParams.MATCH_PARENT, dp(this, 56)));

                      setContentView(root);
                  }

                  private void requestStop() {
                      KeyguardManager keyguardManager =
                              (KeyguardManager) getSystemService(KEYGUARD_SERVICE);
                      if (keyguardManager != null && keyguardManager.isKeyguardLocked()) {
                          keyguardManager.requestDismissKeyguard(this,
                                  new KeyguardManager.OnDismissKeyguardFinishedListener() {
                                      @Override
                                      public void onDismissSucceeded() {
                                          stopAlarm();
                                      }

                                      @Override
                                      public void onDismissCancelled() {
                                      }
                                  });
                      } else {
                          stopAlarm();
                      }
                  }

                  private void stopAlarm() {
                      AlarmService.stop(this);
                      finish();
                  }

                  @Override
                  public void onBackPressed() {
                  }
              }
          }
          EOF

      - name: Set up Gradle
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.7'

      - name: Build APK
        run: gradle assembleDebug

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: feishu-alarm-apk
          path: app/build/outputs/apk/debug/app-debug.apk

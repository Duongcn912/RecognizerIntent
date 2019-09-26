package com.android.egoo.ui.home.conversations;

import android.Manifest;
import android.content.Context;
import android.content.DialogInterface;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.graphics.Color;
import android.media.AudioManager;
import android.media.MediaPlayer;
import android.os.Bundle;
import android.speech.RecognitionListener;
import android.speech.RecognizerIntent;
import android.speech.SpeechRecognizer;
import android.support.annotation.NonNull;
import android.support.design.widget.CoordinatorLayout;
import android.support.design.widget.Snackbar;
import android.support.v4.app.ActivityCompat;
import android.support.v4.content.ContextCompat;
import android.support.v7.widget.LinearLayoutManager;
import android.support.v7.widget.RecyclerView;
import android.support.v7.widget.Toolbar;
import android.util.Log;
import android.util.TypedValue;
import android.view.Menu;
import android.view.MenuInflater;
import android.view.MenuItem;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

import com.android.egoo.R;
import com.android.egoo.adapters.AdapterConversation;
import com.android.egoo.model.Message;
import com.android.egoo.ui.base.BaseActivity;
import com.android.egoo.ui.home.vocabulary.VocabularyActivity;
import com.android.egoo.util.Define;
import com.android.egoo.util.Setting;
import com.android.egoo.util.UtilPref;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

import butterknife.BindView;
import butterknife.ButterKnife;
import cn.pedant.SweetAlert.SweetAlertDialog;

public class ConversationActivity extends BaseActivity implements ConversationMvpView, RecognitionListener {
    public static final int REQUEST_VOICE = 100;
    public static final int TIME_OUT_VOICE = 3000;
    public final String TAG = "ConversationActivity123";
    @BindView(R.id.rcv_message)
    RecyclerView mRcvMessage;
    @BindView(R.id.coordinatorLayout)
    CoordinatorLayout mCoordinator;
    @BindView(R.id.toolbar)
    Toolbar mToolbar;
    @BindView(R.id.btn_practice)
    Button btnPractice;
    private Snackbar mSnackbar;
    private TextView mTvRecommend;
    private AdapterConversation mAdapter;
    private List<Message> mMessages;
    private List<Message> mData;
    private ConversationPresenter mPresenter;
    private String idCategory;
    private String idUnit;
    private int mPostition;

    //voice
    private SpeechRecognizer mSpeechRecognizer;
    private Intent mSpeechRecognizerIntent;
    private boolean isDone;
    private int mScore;
    private int mPreviousScore = 0;
    private int mCount;
    private boolean isRecording = false;
    AudioManager mAudioManager = null;
    //
    Menu menu;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_conversation);
        ButterKnife.bind(this);
        mPresenter = new ConversationPresenter(getBaseContext());
        mPresenter.attachView(this);
        initView();
        initData();
    }


    private void initView() {
        initToolbar();
        mRcvMessage.setLayoutManager(new LinearLayoutManager(this, LinearLayoutManager.VERTICAL, false));
        mMessages = new ArrayList<>();
        mAdapter = new AdapterConversation(mMessages, getBaseContext());
        mRcvMessage.setAdapter(mAdapter);

        btnPractice.setVisibility(View.GONE);
        btnPractice.setOnClickListener(view -> repeatConversation());
    }

    private void initSpeechToText() {
        mSpeechRecognizer = SpeechRecognizer.createSpeechRecognizer(getBaseContext());
        mSpeechRecognizer.setRecognitionListener(this);

        mSpeechRecognizerIntent = new Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH);
        mSpeechRecognizerIntent.putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL,
                RecognizerIntent.LANGUAGE_MODEL_FREE_FORM);
        mSpeechRecognizerIntent.putExtra(RecognizerIntent.EXTRA_LANGUAGE_PREFERENCE, "en");
        mSpeechRecognizerIntent.putExtra(RecognizerIntent.EXTRA_SPEECH_INPUT_COMPLETE_SILENCE_LENGTH_MILLIS, TIME_OUT_VOICE);
        mSpeechRecognizerIntent.putExtra(RecognizerIntent.EXTRA_LANGUAGE, "en");
        mSpeechRecognizerIntent.putExtra(RecognizerIntent.EXTRA_CALLING_PACKAGE, this.getPackageName());
    }

    private void initToolbar() {
        mToolbar.setTitleTextColor(Color.WHITE);
        setSupportActionBar(mToolbar);
        getSupportActionBar().setTitle(Setting.getInstance().getUnit().getTitle());
        getSupportActionBar().setDisplayHomeAsUpEnabled(true);
        getSupportActionBar().setDisplayShowHomeEnabled(true);
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        MenuInflater inflater = getMenuInflater();
        inflater.inflate(R.menu.menu_conversation, menu);
        this.menu = menu;
        return true;
    }

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        // Handle item selection
        switch (item.getItemId()) {
            case android.R.id.home:
                finish();
                return true;
            case R.id.action_vocabulary:
//                new DialogVocabulary(ConversationActivity.this).show();
                startActivity(new Intent(this, VocabularyActivity.class));
                return true;
            default:
                return super.onOptionsItemSelected(item);
        }
    }

    private boolean checkPermission() {
        int hasRecordPermission = ContextCompat.checkSelfPermission(this, Manifest.permission.RECORD_AUDIO);
        if (hasRecordPermission != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this,
                    new String[]{Manifest.permission.RECORD_AUDIO},
                    REQUEST_VOICE);
            return false;
        } else {
            return true;
        }

    }

    private void initData() {
        //demo
        mPostition = 0;
        idCategory = Setting.getInstance().getCategory().getId();
        idUnit = Setting.getInstance().getUnit().getId();
        //
        mPresenter.getConversation(idCategory, idUnit);
    }

    private void startConversation() {
        isDone = false;
        if (checkPermission()) {
            if (mData.size() > 0) {
                mAudioManager.setStreamMute(AudioManager.STREAM_MUSIC, false);
                //init data
                showRecommend("");
                mAdapter.insert(mData.get(mPostition));
//                mAdapter.addMessage(mData.get(mPostition));
                mRcvMessage.smoothScrollToPosition(mPostition);
                mPresenter.playMp3(mMessages.get(mPostition).getAudio());
            }
        }
    }

    private void repeatConversation() {
        if (isDone) {
            mPostition = 0;
            mScore = 0;
            mCount = 0;
            mAdapter.removeAll();
            mAdapter.notifyDataSetChanged();
            startConversation();
            btnPractice.setVisibility(View.GONE);
        }
    }

    @Override
    public void onRequestPermissionsResult(int requestCode,
                                           @NonNull String permissions[], @NonNull int[] grantResults) {
        switch (requestCode) {
            case REQUEST_VOICE: {
                if (grantResults.length == 0
                        || grantResults[0] !=
                        PackageManager.PERMISSION_GRANTED) {
                    Log.i(TAG, "Permission has been denied by user");
                } else {
                    Log.i(TAG, "Permission has been granted by user");
                    startConversation();
                }
            }
        }
    }

    @Override
    public void getConversation(List<Message> mes) {
        mData = mes;
        mMessages.clear();
        if (isForeground) {
            startConversation();
        }
    }

    @Override
    public void onNext() {
        isRecording = false;
        mAdapter.setRecording(isRecording);
        mPostition++;
        if (mPostition < mData.size()) {
            if (mPreviousScore == 100) {
                mAudioManager.setStreamMute(AudioManager.STREAM_MUSIC, false);
                mPreviousScore = 0;
                mPresenter.playCorrectSound();
                try {
                    Thread.sleep(400);
                    nextMessage();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            } else {
                nextMessage();
            }
        } else {
            if (mPreviousScore == 100) {
                mAudioManager.setStreamMute(AudioManager.STREAM_MUSIC, false);
                mPreviousScore = 0;
                mPresenter.playCorrectSound();
                try {
                    Thread.sleep(400);
                    finishConversation();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            } else {
                finishConversation();
            }
        }
    }

    private void nextMessage() {
        if (mData.get(mPostition).getIsRobot()) {
            mAudioManager.setStreamMute(AudioManager.STREAM_MUSIC, false);
            mAdapter.insert(mData.get(mPostition));
            mRcvMessage.smoothScrollToPosition(mPostition);
            mPresenter.playMp3(mData.get(mPostition).getAudio());
            showRecommend("");
        } else {
            mAudioManager.setStreamMute(AudioManager.STREAM_MUSIC, true);
            mSpeechRecognizer.startListening(mSpeechRecognizerIntent);
            isRecording = true;
        }
    }

    private void finishConversation() {
        isDone = true;
        mScore = mScore / mCount;
        mScore = Math.min(mScore,100);
        mAdapter.setRecording(false);
        mPresenter.pushScore(idUnit, UtilPref.getInt(this, Define.ID_USER, 0), mScore);
    }

    @Override
    protected void onStart() {
        super.onStart();
        if (mAudioManager == null) {
            mAudioManager = (AudioManager) getApplicationContext().getSystemService(Context.AUDIO_SERVICE);
            mAudioManager.setStreamMute(AudioManager.STREAM_MUSIC, false);
        }
        initSpeechToText();
        if (!isDone) {
            if (mMessages != null && mPostition > 0 && mPostition < mData.size()) {
                //get status last item
                mPostition = mAdapter.getItemCount() - 1;
                boolean isRobot = mData.get(mPostition).getIsRobot();
                if (!isRobot) {
                    mPostition--;
                }
            }
            if (mData != null) {
                if (mPostition == 0 && mMessages.size() == 0) {
                    startConversation();
                } else {
                    onNext();
                }
            }
        }
    }

    @Override
    protected void onStop() {
        super.onStop();
        mSpeechRecognizer.stopListening();
        mSpeechRecognizer.destroy();
        isRecording = false;
        mPresenter.onStop();
        mAudioManager.setStreamMute(AudioManager.STREAM_MUSIC, false);
    }

    @Override
    public void onReadyForSpeech(Bundle params) {
        Log.d(TAG, "ready");
        showRecommend(mData.get(mPostition).getRecommend());
        mAdapter.setRecording(isRecording);
        mRcvMessage.smoothScrollToPosition(mAdapter.getItemCount());
    }

    @Override
    public void onBeginningOfSpeech() {
        Log.d(TAG, "start");
        isRecording = true;
    }

    @Override
    public void onRmsChanged(float rmsdB) {
    }

    @Override
    public void onBufferReceived(byte[] buffer) {
        Log.d(TAG, "reciever");
    }

    @Override
    public void onEndOfSpeech() {
        Log.d(TAG, "end");
        mAdapter.setRecording(isRecording);
        mSpeechRecognizer.stopListening();
    }

    @Override
    public void onError(int error) {
        //
        switch (error) {
            case SpeechRecognizer.ERROR_AUDIO:
                showLog("ERROR_AUDIO");
                break;
            case SpeechRecognizer.ERROR_CLIENT:
                showLog("ERROR_CLIENT");
                mSpeechRecognizer.stopListening();
                mSpeechRecognizer.cancel();
                mSpeechRecognizer.destroy();
                mSpeechRecognizer = SpeechRecognizer.createSpeechRecognizer(getBaseContext());
                mSpeechRecognizer.setRecognitionListener(this);
                mSpeechRecognizer.startListening(mSpeechRecognizerIntent);
                return;
            case SpeechRecognizer.ERROR_RECOGNIZER_BUSY:
                showLog("ERROR_RECOGNIZER_BUSY");
                return;
            case SpeechRecognizer.ERROR_INSUFFICIENT_PERMISSIONS:
                showLog("ERROR_INSUFFICIENT_PERMISSIONS");
                break;
            case SpeechRecognizer.ERROR_NETWORK_TIMEOUT:
                showLog("ERROR_NETWORK_TIMEOUT");
                break;
            case SpeechRecognizer.ERROR_NETWORK:
                showLog("ERROR_NETWORK");
                break;
            case SpeechRecognizer.ERROR_SERVER:
                showLog("ERROR_SERVER");
                break;
            case SpeechRecognizer.ERROR_NO_MATCH:
                showLog("ERROR_NO_MATCH");
                break;
            case SpeechRecognizer.ERROR_SPEECH_TIMEOUT:
                showLog("ERROR_SPEECH_TIMEOUT");
                break;
            default:
                return;
        }
        mSpeechRecognizer.stopListening();
        mSpeechRecognizer.cancel();

//        mSpeechRecognizer.startListening(mSpeechRecognizerIntent);
//        isRecording = true;

        Log.d(TAG, "error " + error);
    }

    @Override
    public void onResults(Bundle results) {
        Log.d(TAG, "result");
        ArrayList<String> matches = results.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION);
        if (matches != null) {
            String textUser = matches.get(0);
            Message mes = mData.get(mPostition);
            mPreviousScore = Math.max(getScore(textUser, mes.getContext()), getScore(textUser, mes.getRecommend()));
            mScore += mPreviousScore;
            mCount++;
            Log.d(TAG, "Score " + mScore);
            mes.setContext(textUser);
            mAdapter.addMessage(mes);
            mRcvMessage.smoothScrollToPosition(mPostition);
            try {
                Thread.sleep(250);
                onNext();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    @Override
    public void onPartialResults(Bundle partialResults) {
        Log.d(TAG, "res");
    }

    @Override
    public void onEvent(int eventType, Bundle params) {
        Log.d(TAG, "event");
    }

    private void showRecommend(String recommend) {
        if (!recommend.equals("")) {
            recommend = "Recommend: " + recommend;
        }
        if (mTvRecommend == null) {
            mSnackbar = Snackbar
                    .make(mCoordinator, recommend, Snackbar.LENGTH_INDEFINITE);
            mSnackbar.getView().setBackgroundColor(ContextCompat.getColor(this, R.color.colorPrimary));
            mTvRecommend = (mSnackbar.getView()).findViewById(android.support.design.R.id.snackbar_text);
            mTvRecommend.setTextSize(TypedValue.COMPLEX_UNIT_PX, getResources().getDimension(R.dimen.text_size_medium));
            mSnackbar.show();
        } else {
            mTvRecommend.setText(recommend);
        }

    }

    private void showLog(String err) {
        Log.d(TAG, err);
    }

    private int getScore(String user_text_input, String recommend_text_input) {
        String user_text = user_text_input.toLowerCase().trim().replace("'", "");
        String recommend_text = recommend_text_input.toLowerCase().trim();
        recommend_text = recommend_text.replace("?", "")
                .replace(".", "")
                .replace("’", "")
                .replace(",", "")
                .replace("!", "")
                .replace("'", "");
        user_text = user_text.replace("?", "")
                .replace(".", "")
                .replace(",", "")
                .replace("’", "")
                .replace("!", "")
                .replace("'", "");
        int corrected_words_num = 0;
        String[] user_text_arr = user_text.split(" ");
        String[] recommend_text_arr = recommend_text.split(" ");
        for (String anUser_text_arr : user_text_arr) {
            if (Arrays.asList(recommend_text_arr).contains(anUser_text_arr)) {
                corrected_words_num++;
            }
        }
        corrected_words_num = Math.min(corrected_words_num,recommend_text_arr.length);
        return corrected_words_num * 100 / recommend_text_arr.length;
    }

    @Override
    public void showDialogResult() {
        isDone = true;
        btnPractice.setVisibility(View.VISIBLE);
        MediaPlayer mediaPlayer = MediaPlayer.create(this,R.raw.clap);
        mediaPlayer.start();
        SweetAlertDialog alertDialog = new SweetAlertDialog(this, SweetAlertDialog.CUSTOM_IMAGE_TYPE)
                .setTitleText(getString(R.string.dialog_score_title))
                .setContentText(getString(R.string.dialog_score_content)+" " + mScore)
                .setCancelText(getString(R.string.dialog_score_cancel))
                .setConfirmText(getString(R.string.dialog_score_redo))
                .setCustomImage(R.drawable.ic_result)
                .setCancelClickListener(sDialog -> sDialog.cancel())
                .setConfirmClickListener(sweetAlertDialog -> {
                    sweetAlertDialog.dismiss();
                    repeatConversation();
                });
        alertDialog.setOnDismissListener(new DialogInterface.OnDismissListener() {
                    @Override
                    public void onDismiss(DialogInterface dialog) {
                        mediaPlayer.stop();
                        mediaPlayer.release();
                    }
                });
        alertDialog.show();
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        mPresenter.onStop();
        mSpeechRecognizer.destroy();
    }
}

# Mục lục
1. [Giới thiệu sản phẩm](#1)
2. [Các module chính](#2)
3. [Phương pháp - ngôn ngữ - công cụ](#3)
4. [Giới thiệu các hình ảnh kết quả](#4)
5. [Danh sách các tài liệu tham khảo](#5)
6. [Khảo sát](#6)

# 1.Giới thiệu sản phẩm <a name="1"></a>
* Tên sản phẩm: Notes App
* Ngôn ngữ: Kotlin
* Công nghệ: Android
* Link download: [NotesApp.apk](https://drive.google.com/file/d/1u7Upv1raYiUp8XmWvrrpSxvqKqyrqYNB/view)
* Link project: [NotesApp](https://github.com/Sprinklotad/NotesApp)
# 2. Các module chính <a name="2"></a>
* Sử dụng room database để lưu trữ và truy vấn dữ liệu.
* Sử dụng Recycler View để hiển thị các note.
* Sử dụng Dialog, PopUpMenu để hiển thị các bảng chọn.
* Sử dụng Bundle để truyền dữ liệu giữa các Fragment.
* Sử dụng TimePickerDialog và DatePickerDialog để có thể chọn deadline cho note (công việc).
* Sử dụng CalendarView để hiển thị lịch và các công việc có trong ngày.
* Có chức năng Swipe note để xoá và recover note.
# 3. Phương pháp - ngôn ngữ - công cụ <a name="3"></a>
* Ứng dụng được viết trên Android Studio, design giao diện, các màn hình và layout trên file .xml và viết logic bằng ngôn ngữ Kotlin.
* Database có 1 bảng lưu trữ note. Note có các thuộc tính: id, title, description, backGroundColor, isDeleted, timeRemaining, date, time.

    ```kotlin
    @Entity(tableName = "notesTable")
    class Note(
        val noteTitle: String,
        val noteDescription: String,
        val backGroundColor : Int,
        var isDeleted: Int,
        val timeRemaining: Long,
        val time: String,
        val date: String
    ) {
        @PrimaryKey(autoGenerate = true)
        var id = 0
    }
    ```

    - Thuộc tính `title`, `description` để hiển thị các `note` trên màn hình chính.
    - Thuộc tính `backGroundColor` có thể được chọn bởi người dùng, nếu không thì sẽ là màu mặc định.
    - `Note` có thể bị xoá, và sau khi bị xoá thì thuộc tính `isDeleted` = `true` và `note` được chuyển sang màn hình đã xoá (thùng rác). Sau khi xoá ở màn hình thùng rác thì `note` mới bị xoá hẳn.
    - Thuộc tính `date`, `time` là `deadline` cho công việc mà người dùng có thể chọn cho `note`. Từ đó tính ra `timeRemaining` để có thể hiển thị trên màn hình và `sort` theo thời gian còn lại. Thuộc tính `date` cũng để hiển thị `note` theo ngày trên màn hình lịch.
* Các màn hình có sử dụng `RecyclerView` để hiển thị các `note` có thể cuộn.
    - `Adapter`:

        ```kotlin
        class NoteRVAdapter(
            val context: NotesFragment,
            private val noteClickInterface: NoteClickInterface
        ) : RecyclerView.Adapter<NoteRVAdapter.ViewHolder>() {
        
            private val allNotes = ArrayList<Note>()
        
            inner class ViewHolder(binding: NoteRvItemBinding) :
                RecyclerView.ViewHolder(binding.root) {...}
        
            override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {...}
        
            override fun onBindViewHolder(holder: ViewHolder, position: Int) {...}
        
            private fun getTimeRemain(date: String, time: String): String {...}
        
            fun getNoteAt(position: Int): Note {...}
        
            override fun getItemCount(): Int {...}
        
            fun updateList(newList: List<Note>) {...}
        }
        ```

    - Set up `Adapter` trong `Fragment`.

        ```kotlin
        private fun setUpAdapter() {
            noteRV.layoutManager = LinearLayoutManager(context)
            noteRVAdapter = NoteRVAdapter(this, this)
            noteRV.adapter = noteRVAdapter
            viewModel = ViewModelProvider(this)[NoteViewModel::class.java]
            listNote = viewModel.listNote
            listNote.observe(viewLifecycleOwner) { list ->
                noteRVAdapter.updateList(list)
            }
        }
        ```

* Sử dụng `SQLite` để truy vấn các `note` cho màn hình hiển thị `note`, màn hình thùng rác, search note và search ngày theo lịch hiển thị.

    ```kotlin
    @Query("Select * from notesTable Where isDeleted = 0")
    fun getListNotes(): LiveData<List<Note>>
    @Query("Select * from notesTable Where isDeleted = 1")
    fun getDeletedNotes(): LiveData<List<Note>>
    @Query("SELECT * FROM notesTable WHERE noteTitle LIKE :title AND isDeleted = 0")
    fun searchByTitle(title: String): LiveData<List<Note>>
    @Query("SELECT * FROM notesTable WHERE date LIKE :mDate AND isDeleted = 0")
    fun calendarSearch(mDate: String): LiveData<List<Note>>
    ```

* Ứng dụng có sử dụng thư viện `com.github.sundeepk:compact-calendar-view:3.0.0` để hiển thị màn hình lịch với tính thẩm mỹ hơn và có thêm chức năng hiển thị các dấu `note` ở các ngày có công việc trên màn hình lịch.

    Set up `CalendarView`: Có phương thức `click` vào từng ngày để hiển thị các công việc và phương thức cuộn theo tháng để xem các tháng khác.

    ```kotlin
    private fun setUpCalendar() {
        val sdf = SimpleDateFormat("dd.MM.yyyy", Locale.getDefault())
        val currentDate: String = sdf.format(Date())
        binding.textMonth.text =
            SimpleDateFormat("MMMM - yyyy", Locale.getDefault()).format(Date())
        binding.calendarView.shouldDrawIndicatorsBelowSelectedDays(true)
        showCalendarItemRecyclerView(currentDate)
        binding.calendarView.setListener(object : CompactCalendarView
        .CompactCalendarViewListener {
            override fun onDayClick(date: Date) {
                val searchQuery = sdf.format(date)
                showCalendarItemRecyclerView(searchQuery)
            }
            override fun onMonthScroll(date: Date) {
                binding.textMonth.text =
                    SimpleDateFormat("MMMM - yyyy", Locale.getDefault()).format(date)
                val searchQuery = sdf.format(date)
                showCalendarItemRecyclerView(searchQuery)
            }
        })
    }
    ```

* Sử dụng thư viện `com.github.xabaras:RecyclerViewSwipeDecorator:1.4` để chức năng `Swipe note` tốt hơn.

    `Set up Swipe`: Khi `swipe note` trong màn hình chính thì sẽ xoá note, có background và có thể undo.

    ```kotlin
    private fun setUpSwipe() {
        ItemTouchHelper(object :
            ItemTouchHelper.SimpleCallback(0, ItemTouchHelper.LEFT or ItemTouchHelper.RIGHT) {
            override fun onMove(
                recyclerView: RecyclerView,
                viewHolder: RecyclerView.ViewHolder,
                target: RecyclerView.ViewHolder
            ): Boolean {...}
            override fun onSwiped(viewHolder: RecyclerView.ViewHolder, direction: Int) {...}
            override fun onChildDraw(
                c: Canvas, recyclerView: RecyclerView, viewHolder: RecyclerView.ViewHolder,
                dX: Float, dY: Float, actionState: Int, isCurrentlyActive: Boolean
            ) {...}
        }).attachToRecyclerView(binding.idRVNotes)
    }
    ```
    
# 4. Giới thiệu các hình ảnh kết quả <a name="4"></a>
* Màn hình chính:

    <img src="https://user-images.githubusercontent.com/84316258/203465136-f0bce853-5ff7-45bd-8e2f-198a37eba153.jpg" width="200" />
    <img src="https://user-images.githubusercontent.com/84316258/203465390-da996022-b7f5-4b30-a9bd-dd67527840aa.png" width="200" />

* Khi ấn nút `+` (`FloatingActionButton`) thì ứng dụng sẽ chuyển qua màn hình tạo mới `note`.

    <img src="https://user-images.githubusercontent.com/84316258/203465407-159ea5a6-4c56-4756-b4df-a0d454b2c706.jpg" width="200" />
    <img src="https://user-images.githubusercontent.com/84316258/203465417-a28d5c15-3ac5-4733-a159-288ea8a1cfaa.png" width="200" />

    Hàng trên cùng là các ImageView gồm:
    -	Nút back để quay trở lại màn hình chính.
    -	Bảng chọn màu để chọn BackGroundColor cho note.
    -	TimePicker và DatePicker để chọn Deadline cho note.
    -	Nút Tick để kết thúc chỉnh sửa và ẩn KeyBoard.

* Khi click vào bảng chọn màu, màn hình sẽ hiển thị `Dialog` để chọn `BackGroundColor` cho `note`.

    <img src="https://user-images.githubusercontent.com/84316258/203465428-a448b26d-2bdd-4fc2-9147-9e9901fea1f9.jpg" width="200" />
    <img src="https://user-images.githubusercontent.com/84316258/203465437-3e9a0677-e73c-4c0b-925b-a3864651e9fa.png" width="200" />

* `TimePicker` và `DatePicker`.

    <img src="https://user-images.githubusercontent.com/84316258/203465444-80fc8093-3c1a-44fe-8217-e2b94ee85bdc.jpg" width="200" />
    <img src="https://user-images.githubusercontent.com/84316258/203465451-3ca4ed17-4ee1-494f-a51d-78781abd0ce6.jpg" width="200" />

* Màn hình khi hiển thị các `notes`.

    <img src="https://user-images.githubusercontent.com/84316258/203465463-5c47e7de-1a16-49f7-a9c4-8e22dad39816.jpg" width="200" />

* Khi click vào icon `sort`, màn hình sẽ hiện lên 1 `PopUpMenu` để có thể sort note theo `id`, `title` hoặc `timeRemaining`.

    <img src="https://user-images.githubusercontent.com/84316258/203465468-9800e7f3-e1de-411c-ad05-5941ab596ecc.jpg" width="200" />
    <img src="https://user-images.githubusercontent.com/84316258/203465475-e61594f9-2515-4fda-9361-a0b0672eac4b.png" width="200" />

* Sau khi sort theo title.

    <img src="https://user-images.githubusercontent.com/84316258/203465484-2d187ea0-0899-407f-96c4-ca64fc99c2e1.jpg" width="200" />

* Khi chuyển sang màn hình `Calendar`, sẽ hiển thị 1 lịch theo tháng. Những ngày có công việc sẽ có các dấu chấm ở dưới ngày.

    <img src="https://user-images.githubusercontent.com/84316258/203465490-59ce1a49-f888-4572-a50a-5f52df719bfe.jpg" width="200" />

* Khi chọn ngày có công việc, các công việc sẽ hiển thị theo thời gian.

    <img src="https://user-images.githubusercontent.com/84316258/203465502-c94c656b-ba66-41ef-bc82-a594df296d64.jpg" width="200" />

* Có thể vuốt note sang ngang để xoá note và có thể undo.

    <img src="https://user-images.githubusercontent.com/84316258/203465510-2350d311-b697-4a5f-90ac-c4c2f26784ca.png" width="200" />
    <img src="https://user-images.githubusercontent.com/84316258/203465517-1c2f6d90-33fc-43eb-9238-6b22ea7415f1.png" width="200" />

* Sau khi note được xoá thì sẽ được chuyển qua thùng rác. Note ở đây có thể được xoá hẳn hoặc `recover`.

# 5. Danh sách tài liệu tham khảo <a name="5"></a>

* [Room database](https://developer.android.com/jetpack/androidx/releases/room?gclid=Cj0KCQiAsdKbBhDHARIsANJ6-jftHAcrUwPPDsT7sPNk9SY95-6YpYsB59nY-KrGL53L-0UCHqeDiJ8aAhHYEALw_wcB&gclsrc=aw.ds)
* [RecyclerView](https://developer.android.com/develop/ui/views/layout/recyclerview)
* [Swipe recyclerView](https://github.com/xabaras/RecyclerViewSwipeDecorator)
* [CalendarView](https://github.com/SundeepK/CompactCalendarView)
* [TimePicker](https://developer.android.com/reference/android/widget/TimePicker)
* [DatePicker](https://developer.android.com/reference/android/widget/DatePicker)
# 6. Khảo sát <a name="6"></a>


<img src="https://user-images.githubusercontent.com/84316258/203467416-78cba39b-3901-43f1-949e-b77a314f310a.png" width="600" />
<img src="https://user-images.githubusercontent.com/84316258/203467428-15eaab4a-0ae7-4793-ac10-c40923b5720c.png" width="600" />
<img src="https://user-images.githubusercontent.com/84316258/203467433-90611372-c282-4da0-94ed-a03269a5a768.png" width="600" />
<img src="https://user-images.githubusercontent.com/84316258/203467714-3338a5fd-b71f-40ab-8f1f-0ec11228b4e2.png" width="600" />
<img src="https://user-images.githubusercontent.com/84316258/203467719-75d21425-8ed5-44ca-8628-74cbf97ab0a8.png" width="600" />
<img src="https://user-images.githubusercontent.com/84316258/203467723-01852e15-d606-4e7f-99c7-5f330e56c67a.png" width="600" />
<img src="https://user-images.githubusercontent.com/84316258/203467733-7756cf78-50f0-40a0-938b-d1a288d811f9.png" width="600" />
<img src="https://user-images.githubusercontent.com/84316258/203467746-3b27e410-52b3-4c41-9d60-4651fd3721d7.png" width="600" />
<img src="https://user-images.githubusercontent.com/84316258/203467754-a667dc3b-b8f6-400f-b9bd-bbb1cd68c4ae.png" width="600" />
<img src="https://user-images.githubusercontent.com/84316258/203467761-2d632e73-9417-43a0-bfec-d5246d81cfc5.png" width="600" />

# QR khảo sát

<img src="https://user-images.githubusercontent.com/84316258/203468046-70a24a00-3a66-4313-813c-63b5fa7a4a45.png" width="600" />

# QR Project

<img src="https://user-images.githubusercontent.com/84316258/203468388-1abd1856-5024-4404-b9fd-bf15e85edeb9.png" width="600" />
